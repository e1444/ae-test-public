# Summary: Support & Customer Communications

**Source:** `corpus/06_support_customer_comms.md` — transcript, Mar 7 2025. Attendees: Amara Diallo
(Head of Support), Marcus Webb (Support Lead), Chloe Baptiste (Marketing), Elena Sokolov (Eng).

## Overview
Support's walkthrough of the customer-facing "blast radius" of the migration: statement descriptor
changes, cards that fail to migrate, self-serve payment flows, 3D Secure behavior for EU cards,
mid-dunning notification collisions, Founders-specific care, and rollout pacing. Frames rollout
speed explicitly as a support decision, not purely an engineering one.

## Stakeholder decisions
- **Amara Diallo:** Whatever the new statement descriptor is, the decision and customer
  communication must happen **before** any Stripe charge appears with it — today's descriptor
  (`ATLASPAY*STREAMHAVEN`) is already muscle-memory for ~80% of customers, and routine descriptor
  glitches already generate ~40 tickets/month baseline.
- **Chloe Baptiste:** Marketing's preference is a clean `STREAMHAVEN` descriptor with no processor
  prefix; final call depends on engineering-supplied technical constraints (character limits,
  processor prefix conventions) — engineering brings options, Chloe/marketing picks.
- **Elena Sokolov:** Commits that whatever descriptor is chosen, comms must precede the first
  charge under it (agreement, not just aspiration).
- **Amara Diallo:** Cards that fail migration need a designed "your card on file couldn't be moved —
  update it here" journey (comms + working self-serve update link) — not just silent reliance on
  Stripe's own dunning retry.
- **Marcus Webb:** The update-card link must work on mobile and clearly identify which subscription
  it refers to — named as the majority root cause of today's update-card tickets.
- **Amara Diallo:** Self-serve add/replace-card and cancel flows must keep working once Stripe
  becomes system of record — someone must own redesigning or hosting-equivalent-configuring them;
  Amara's macro library must match whatever ships (scoping problem, not an invention problem).
- **Elena Sokolov:** Commits the PRD will take an explicit position on 3DS behavior (authentication
  request configuration, non-completion handling, whether to ask customers to pre-authorize ahead of
  cutover) — and that EU challenge-ticket volume should be assumed to rise in week one regardless.
- **Marcus Webb / Chloe Baptiste:** Pre-cutover, pre-warm the EU customer base with a one-line
  "your bank may ask you to approve your next payment" email — added to the comms matrix.
- **Amara Diallo:** Whoever owns the dunning-treatment decision for the 6 mid-cycle customers must
  also own designing which system's notifications those customers receive — explicitly paired so a
  customer never gets two different systems' "payment failed" emails in the same week.
- **Marcus Webb:** Strong preference for a **phased rollout** (e.g., over two weeks with a checkpoint
  after each batch) over a single "big bang" cutover of all 100 customers — so descriptor, failed-
  card, and 3DS ticket spikes don't all land the same morning; explicitly says he'd trade rollout
  speed for load control.
- **Amara Diallo:** Rollout pacing is called out as a **support decision**, not purely engineering's,
  and belongs in the PRD.
- **Amara Diallo / Chloe Baptiste:** Founders cohort (8 customers, since 2005) gets a dedicated,
  personally-toned comms track — three specifically are known to call about everything, two
  reconcile statements by hand.

## What must stay the same
- Self-serve payment management capability (add/replace card, cancel) must continue to exist in
  some form post-cutover — only the underlying implementation may change.
- Customers must hear about changes (descriptor, card issues, 3DS) from Streamhaven proactively,
  never discover them via an unexplained bank line or unexplained failed-charge email.

## What must change
- Statement descriptor will change from `ATLASPAY*STREAMHAVEN` to something new (exact string still
  undecided — options to be brought by engineering).
- Failed-payment/dunning handling and notifications move to Stripe's tooling for anyone not covered
  by the (still-undecided) legacy-dunning handling plan.
- 3DS/SCA authentication flow moves to Stripe/issuer-driven challenges — needs an explicit
  configuration decision and a customer-facing explanation for challenge emails.
- Self-serve card/cancel UI flows need to be rebuilt or reconfigured against Stripe as the new
  system of record.
- Rollout must be batched/phased (not all 100 customers at once) with checkpoints between batches.

## Open questions raised (for PRD)
1. Final statement descriptor string (character-limit/prefix constraints vs. `STREAMHAVEN`-clean
   preference) — decide and pre-communicate before first charge.
2. Design of the failed-card "update your card" journey (comms + self-serve link, mobile-friendly,
   subscription-specific).
3. Scope/ownership of self-serve payment-method and cancellation flow redesign.
4. 3DS/SCA configuration decision + EU pre-cutover heads-up email content.
5. Notification ownership/design for the 6 mid-dunning customers (paired with whoever owns the
   dunning-treatment decision itself).
6. Rollout batch size/cadence and checkpoint criteria between batches.
7. Founders-specific communications track content (separate from general comms matrix).
