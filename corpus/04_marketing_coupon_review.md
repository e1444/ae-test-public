# Transcript: Project Relay — Marketing Coupon Review

**Date:** Thursday, March 6, 2025, 1:00 PM
**Attendees:** Chloe Baptiste (VP Marketing), Luis Herrera (Growth), Jonas Meyer (Eng), Elena
Sokolov (Eng)

---

**Chloe:** Okay — coupons. Elena asked for "everything live, with exact mechanics," so here it is.
But first, a framing: promos are how we hit Q1 numbers. If migration eats even one person's
discount, I hear about it from the customer, who heard about it from Twitter.

**Luis:** Full inventory. Five codes live right now:

**`START25`** — New Year promo. 10% off the **first 5 monthly invoices**. Ran Jan 1 through Feb 28
for new signups. Six subscribers are still inside their window — signed up in January, so they've
consumed one, two, maybe three of their five discounted months. The rest is *owed to them*.

**`FRIEND50`** — referral promo, 50% off **3 monthly invoices**. Four active.

**`STREAM30`** — our always-on monthly push: 30% off, and this one is **monthly billing only — it
does not apply to annual**. Never has. Our engine enforces it at redemption. Five active
subscriptions.

**`TAKE5`** — $5 off a single invoice, one shot. Two active.

**`ANNUAL20`** — 20% off, **forever**, for anyone who signed up on annual during last year's push.
Three subscriptions. It runs until cancelled, which is the nasty one.

**Elena:** Quick mechanics check — when you say START25 gives "10% off the first 5 monthly
invoices," that's five *consecutive* months from signup. Correct?

**Chloe:** Correct. And the customer was told "3 months of 20% off" in email for FRIEND50 — no
wait, FRIEND50 is the 50% one, Luis, the email copy is right on the site?

**Luis:** Site's right. 50% off, first three invoices. Email copy was corrected in December, there's
a stray blog post that says "three months of 20%" — it's wrong, ignore the blog post. Blog is
stale, the site is truth.

**Jonas:** Noted: site is truth. I'll take the inventory back to engineering. One flag before we
move on: several of these subscribers are partway through a discount window — START25 especially.
"Owed to them" is the phrase I'm writing down. Any design that loses track of what each customer
has already consumed versus what's still promised is the one that fails quietly.

**Chloe:** Then write that down in whatever the engineering bible is, because "my discount vanished"
is a churn event. Now — the one that gives me hives. **STREAM30.**

**Luis:** Right. The problem: STREAM30 is monthly-only *in our engine*, and I'm told Stripe coupons
don't do "monthly only."

**Jonas:** The honest version: the monthly-only rule lives in *our* engine — we enforce it at
redemption — and my early read is it doesn't transfer to Stripe as a rule. I want that verified,
not assumed. What I can say today is what we can't do: pretend the old enforcement carried over.
The moment someone applies STREAM30 to an annual sub from a Stripe dashboard, it will happily
discount.

**Chloe:** And I do *not* want annual customers getting 30% off by accident, because annual is
where margin lives. So the PRD needs an explicit answer: how does STREAM30 stay monthly-only after
migration? Whatever mechanism — I need the guarantee, plus what happens to the five current
STREAM30 subscribers. They're monthly, so they're fine *today* — the question is enforcement
tomorrow.

**Luis:** And a second-order question: at cutover, does STREAM30 stay live for *new* signups? It's
evergreen. If it pauses for two weeks during migration, growth notices. I'd rather it keep working
— even if the enforcement is uglier — than pull a live promo.

**Jonas:** Keeping it live is possible, it just has to be a conscious choice in the design. Adding
to the open items: "STREAM30 enforcement + continuity during cutover."

**Chloe:** TAKE5 — the five-bucks-off one. That's five US dollars, yes? Because we have Canadians
and Europeans. If a Toronto customer redeems TAKE5, is it five CAD? Five USD converted? This was
never a problem in our engine because it did one thing, and now—

**Elena:** Now it's a real question. Fixed amounts across three currencies is exactly the kind of
thing that seems trivial until it isn't — "five off" is not automatically the same gesture in
Toronto or Berlin as it is in Ohio. Someone decides what TAKE5 means in each market; I'll work out
the mechanics from the decision. There are only two active TAKE5 redemptions and I believe both
are US, but the coupon definition needs an answer regardless.

**Luis:** For the record my vote is a CAD and EUR equivalent — 5 off locally, it's a marketing
number not an FX trade. But that's Chloe's call to ratify.

**Chloe:** Ratified pending the PRD saying it out loud. Last thing from me: **communications.**
Every migrating customer should hear about this *from us first*. Not from a weird bank line, not
from a failed card email they don't understand. I want a notification plan in the PRD — who gets
told what, and when. And I want the Founders people handled with particular care; they've been with
us since the dial-up era, they *will* call.

**Amara** *(via note, support rep)*: "Since the dial-up era" — three of them literally are. Keep
their email extra gentle.

**Jonas:** Comms plan is a named PRD section. Founders flagged. Anything else?

**Chloe:** One rumor to kill: someone said migration "resets" renewal dates so annuals re-bill
immediately. If that happens to even one customer, the promo program is dead for a year. Renewal
dates preserved — I have it in writing from Priya, but it belongs in the PRD as a hard requirement
with a test around it.

**Elena:** It'll be a hard requirement. If we model the cutover correctly it's structural, not a
patch.

---

*Transcript ends. 38 minutes.*
