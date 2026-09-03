# Transcript: Project Relay — Engineering Design Review

**Date:** Wednesday, March 5, 2025, 10:00 AM
**Attendees:** Sam Okafor (VP Eng), Jonas Meyer (Staff Eng, Billing), Elena Sokolov (Senior Backend
Eng), Wren Tanaka (Head of Data), Amara Diallo (Head of Support, partial)

---

**Sam:** Okay. Design review for Relay. Jonas has the architecture draft. Jonas, walk us through it.

**Jonas:** High level first. We have three internal tables that matter: `customers`,
`subscriptions`, `payment_methods`. One row each per customer right now — a hundred active
subscribers, which sounds tiny, but the shape is what matters, not the count.

**Elena:** A hundred rows? That's not a migration, that's a Tuesday.

**Jonas:** It's a hundred *today* because churn took us down and marketing is re-launching in Q3 —
the engine is built for hundreds of thousands and the new owner expects growth. So we build this
like it's a hundred thousand. Idempotent, resumable, observable. Agreed?

**Sam:** Agreed. Continue.

**Jonas:** Payment facts. We store raw card numbers from signup — yes, we are PCI-scoped, it's on
the audit register, don't email me about it. But we never *bill* on raw cards. At signup the PAN
went to AtlasPay once, they returned a token, and every charge since has been token-only. The
`payment_methods` table holds AtlasPay tokens plus display metadata — brand, last4, expiry.

**Elena:** Which means the PANs have to reach Stripe somehow, or there's no migration. Stripe's card
import is exactly built for this: we put a file on their SFTP, encrypted with PGP, they process it
and send back a response file with new Stripe IDs. I prototyped the file shape against the docs.

**Sam:** Walk me through the flow end to end.

**Elena:** We asked AtlasPay for a vault export — token, customer ref, PAN, expiry. That's the
`atlaspay_card_vault_export` file in the data folder, all one hundred rows. We PGP-encrypt it,
SFTP it to Stripe's migration inbox with the manifest, Stripe creates customers and payment methods
on their side, and drops the response file back on the SFTP. Each row maps our references to a new
`cus_` and `pm_` ID. That's all the file is — an old-to-new mapping, nothing else comes back.

**Amara:** And the failures? Because I guarantee we have some.

**Elena:** Test batch told us something important. Fifteen tokens went in, **twelve came back.**
The file doesn't flag failures — there's no status column, failed cards are just *absent*. You find
them by diffing what you sent against what returned. The three missing were two expired cards and
one the issuing bank had closed; we only know the *why* because we cross-checked AtlasPay's card
records, not because Stripe told us.

**Jonas:** Which is why Wave 1 and Wave 2 are separate. Wave 1 is cards. Nothing customer-facing
changes — we're not charging through Stripe yet. It's pure preparation: Stripe-side customer and
payment method objects exist, keyed back to our IDs.

**Sam:** Keyed back how? This is the part I care about.

**Elena:** That's the open design question, and it runs in two directions: whatever we create on
Stripe's side has to be traceable back to a `CUST-####`, and the new IDs coming back in the
response file need somewhere to live in ours. The PRD has to answer it.

**Wren:** Flagging for my team — however those IDs ride along, Data Pipeline carries them into the
warehouse. That's exactly what I need for reconciliation. I have a separate thread on the rest of
my requirements; I'll circulate it.

**Sam:** Good. Wave 2.

**Jonas:** Wave 2 is where the real decisions are. Products and prices first: two plans, monthly and
annual, three currencies. That's twelve price objects plus the Founders price. Then subscriptions —
and here's the thing. A subscription in Stripe is not a row in our table. Our engine has states
Stripe doesn't have, and Stripe has semantics we don't.

**Sam:** Give me the ugly list.

**Jonas:** One: **dunning**. Our engine retries failed payments on its own schedule — 30-day cycle,
four retries, escalating emails, then forced churn. We have six customers mid-cycle *right now*.
My working assumption is Stripe has no way to import "this customer is on retry three of four with
an email queue running" — but that's an assumption we haven't verified, and either way, what we
*do* about those six is a decision, not a discovery.

**Elena:** We could let them finish their cycle in the old engine and hand them over after—

**Sam:** The old engine is being retired, Elena. That's the entire point of the project. The answer
can't be "the old engine lives on for a subclass of customers." Keep going, Jonas.

**Jonas:** Two: **scheduled downgrades**. We support queued plan changes — someone on Pro schedules
the drop to Basic at period end. It's a row in our table: `pending_plan_change`,
`pending_plan_effective_at`. Five customers are queued right now, all Pro to Basic, all effective in
April. Nobody has worked out what that looks like on Stripe's side — and if we lose those rows in
the move, five people get charged Pro prices after choosing Basic. Support will find out before we
do.

**Amara:** I will personally find out first, yes.

**Jonas:** Three: **discounts**. Five coupons running, and several subscribers are partway through
a discount window — they've consumed some of what they were promised. Marketing will walk you
through the inventory later, so I'll just plant the flag: this is the failure mode that costs us
customers *silently*. Nobody notices for months, and by then it's renewal season. Design
accordingly.

**Sam:** And four?

**Jonas:** Four is the one marketing keeps escalating about, so I'll let them run the details, but
the short version: our `STREAM30` coupon is restricted to monthly billing in our engine, and my
early read is that the restriction doesn't transfer to Stripe as a rule. I want that verified,
not assumed. Marketing owns the outcome; engineering owns making whatever they decide not break.

**Elena:** There's also trials — four customers mid-trial right now. And the annuals: if we get
renewal dates wrong, someone's card gets hit on the wrong day. Priya will notice. The board will
notice.

**Sam:** Renewal dates must be preserved. That's a requirement, not a preference. Priya said
"reconcile to the penny" and she means it.

**Jonas:** Which brings up **the cutover question** — the thing I keep getting asked at lunch. When
is someone actually "live on Stripe"? We know the end state: every subscription exists in Stripe,
we've verified it, and then there's a flip. What the flip looks like — what state things sit in
beforehand, who's allowed to charge whom during it, how a customer gets marked — that's the design
hole in the middle of this project, and it's the one with the worst failure mode.

**Elena:** Mechanically that means our tables change. Ops asked for a dashboard of who bills where —
right now we have nowhere to put that answer. I don't care what we call it; I care that the states
are explicit and that "is this customer billing in Stripe tonight?" is answerable from a query,
not from someone's memory.

**Sam:** So write the state machine in the PRD. Don't hand ops a boolean and a prayer.

**Amara:** Two things from support before we close. One — everyone's bank statement says
"ATLASPAY*STREAMHAVEN" today. If Stripe charges look different, that's a ticket storm. Whatever the
descriptor becomes, I need it decided early and communicated. Two — my team needs the self-serve
flows to survive: add a card, replace a card, cancel. Today those hit our engine. If Stripe is the
system of record, what do those buttons do now? Nobody has answered that.

**Elena:** Options exist at every depth for the customer-facing side — it's scoping, not invention.
But yes, it's in the PRD.

**Sam:** Last one for me: **how does Stripe data get back into Streamhaven?** After cutover,
renewals, failures, cancellations — all of it happens on Stripe's side, and we cannot be blind to
any of it. Our tables, Wren's warehouse, finance's month-end — every consumer needs an answer.
Elena?

**Elena:** That's a genuine design question, not a checkbox. There's more than one consumer and
they don't all have the same tolerance for delay or the same definition of "truth." It's in the
PRD.

**Wren:** Confirming my side is Data Pipeline for the warehouse — roughly T-minus-a-day. My
dashboards can live with that; what they can't live with is our business dimensions going missing.
Thread incoming.

**Sam:** Good. Let's also name what we're **not** deciding today: the tax approach — Priya owns
it, decided before the first Stripe invoice. Dunning policy for the six mid-cycle customers. The
STREAM30 implementation. Exact descriptor string. Those go to the PRD as open items with owners.

**Jonas:** One more constraint for the record: the June renewal wave. Something like a third of our
annuals renew in the first two weeks of June. We cut over *before* that window, with enough soak
time that we're not debugging during someone's renewal.

**Sam:** Wave 1 in the next few weeks, Wave 2 landed with soak time before June. Write it up. And
Jonas — the PRD should assume the reader hasn't sat in this room. Every decision, the alternative,
and why. Thanks everyone.

---

*Transcript ends. 47 minutes.*
