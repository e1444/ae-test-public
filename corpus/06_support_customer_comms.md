# Transcript: Project Relay — Support & Customer Communications

**Date:** Friday, March 7, 2025, 11:00 AM
**Attendees:** Amara Diallo (Head of Support), Marcus Webb (Support Lead), Chloe Baptiste
(Marketing), Elena Sokolov (Eng)

---

**Amara:** I've been in every Relay meeting so far nodding politely. This one is mine. When the
switch happens, my team absorbs the blast. I want to walk through exactly how.

**Elena:** That's why you're on the invite.

**Amara:** Item one: **the statement descriptor.** Today every charge reads `ATLASPAY*STREAMHAVEN`.
Eighty percent of our customers have that string memorized — not consciously, but their bank app
does, their spouse does. If Stripe's charge shows something else, that's a "was I hacked?" ticket.
We get forty of those a *month* from routine descriptor glitches. Show me the plan for what the new
descriptor is and how customers hear about it before it appears.

**Chloe:** On branding I'd fight for `STREAMHAVEN` clean, no processor prefix. It's our name on
their statement, full stop. Whatever the technical constraints are — character limits, processor
prefix conventions — engineering tells me the options and I'll pick.

**Elena:** There *are* constraints; I'll bring options to the PRD. But Amara's right that whatever
we choose, the comms has to precede the charge.

**Amara:** Item two: **cards that fail to migrate.** Elena's test batch had expired and closed
accounts in it. Walk me through the customer experience.

**Elena:** The migration response tells us per-card success or failure. Failed card = no Stripe
payment method = when their renewal hits, there's nothing to charge. Today that flows into our
dunning engine — retry, email, retry, final notice. Post-migration, Stripe's retry and dunning
tools take that job. But a customer whose card expired in November needs a *new card* from us, not
just retries. Someone has to design the "your card on file couldn't be moved — update it here"
journey. It's comms plus a working self-serve update flow on the other end of the link.

**Marcus:** And the link has to work on mobile, and it has to be obvious which subscription it
refers to. Half our update-card tickets are people with an old card and three emails open.

**Amara:** Which is item three: **self-serve payment management has to survive.** Today customers
can add a card, swap a card, and cancel, all without us. Those flows talk to our engine. When
Stripe is the system of record — what do those buttons do? Someone is redesigning those flows or
configuring hosted equivalents, and my macro library needs to match whatever ships.

**Elena:** Options exist at every depth — it's scoping, not invention. But yes, it's in the PRD.

**Amara:** Item four, and this is the Europe one: **3D Secure.** Our EU cards — what, a fifth of
the base? — increasingly get challenged at charge time. The issuer demands 3DS, the customer has to
approve it in their banking app, *then* the charge completes. My questions: does migration change
when 3DS gets triggered? If a renewal happens while the customer is asleep, does the charge fail
and become a ticket? And what do we tell customers in that "your bank needs to approve this"
email?

**Elena:** Short version: 3DS behavior is set by the issuer and the region, and yes, a challenge
can arrive at renewal time. There are real design choices — how we configure authentication
requests, what happens on non-completion, whether we ask customers to pre-authorize ahead of
cutover — and the PRD needs to take a position. Your ticket forecast should assume *some* increase
in EU challenge tickets in week one.

**Marcus:** Can we pre-warm that? Email the EU base before cutover: "your bank may ask you to
approve your next payment." One line, huge ticket reduction.

**Chloe:** Add it to the comms matrix. EU base, pre-cutover, bank-approval heads-up.

**Amara:** Item five: **the mid-dunning customers.** Six people are inside our 30-day retry cycle
right now. Whatever engineering decides — carry them over as failed, let them finish with the old
engine — I need to know *which emails they get*, because today the dunning emails come from us and
after cutover the payment-failure emails might come from Stripe's machinery. If a customer gets
"we couldn't process your payment" from two different systems in one week, that's a trust failure.

**Elena:** Fair. Whoever owns the dunning decision in the PRD also owns the notification design for
it. I'll pair them.

**Amara:** Last item, then I'll stop. The **Founders** people. Eight customers, since 2005, 4.99
forever. Three of them call us about everything. They don't care what Stripe is. If their renewal
date shifts *one day* they will notice, because two of them reconcile their statements by hand —
with a pencil. Whatever the plan is for legacy customers, I want their comms written like it's
going to a person, because it is.

**Chloe:** Founders comms get a dedicated track. Handwritten-adjacent.

**Marcus:** One operational note for whoever writes the rollout plan: **phase my load.** If all
hundred customers switch in one night, descriptor tickets, failed-card tickets, and 3DS tickets all
land the same morning. If it batches over two weeks, my team breathes and we learn from batch one.
I'd take a slower rollout with a checkpoint after each batch over a big bang every day of the week.

**Amara:** Put that in the PRD too. Rollout speed is a support decision, not just an engineering
one. Okay — my list: descriptor plan, failed-card journey, self-serve flows, 3DS expectations,
dunning notifications, Founders care, phased load. That's the blast radius. Cover it and week one
is boring. Boring is the goal.

---

*Transcript ends. 44 minutes.*
