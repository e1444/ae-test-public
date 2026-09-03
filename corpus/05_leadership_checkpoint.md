# Transcript: Project Relay — Leadership Checkpoint

**Date:** Friday, March 7, 2025, 3:00 PM
**Attendees:** Dana Whitfield (CEO), Priya Nair (CFO), Ruth Calloway (Controller), Sam Okafor
(VP Eng)

---

**Dana:** Status, risks, dates. Thirty minutes. Sam, you first.

**Sam:** Design review done. Architecture holds: Wave 1 is customers and cards through Stripe's
secure import — encrypted file over SFTP, we get a mapping file back. Wave 2 is billing: prices,
coupons, subscriptions, cutover. The PRD lands next week with the full plan, decisions, and dates.

**Priya:** I've read the design notes. Two of my lines in the sand are already in there — no double
billing, renewal dates preserved. I want to see them as *mechanisms*, not intentions. How does the
system guarantee it, physically?

**Sam:** That's the core of the PRD, and you'll get mechanisms next week, not intentions.

**Priya:** Good. Then my topic: **authorization rates.** Board-level metric. We're at 91.4% blended
last quarter — 93 in North America, 88 in Europe where 3DS eats us alive. Three risks as I see it.
One: every card in Stripe is a card Stripe has never seen charge. First charge out of the gate is
the riskiest it will be all year. Two: our retry infrastructure resets — the in-house dunning
engine knows these six mid-cycle customers intimately; Stripe's retry machinery knows nothing about
them. Three: descriptor change plus re-auth plus new processor equals a fraud-panic cocktail in
support queues.

**Ruth:** I'll add four: reconciliation. First two months post-cutover I tie every cent out. If
Stripe and our engine disagree about a charge that happened *during* the switch, I need one source
of truth written down, not a meeting.

**Sam:** All four go in the PRD, auth rate first among them. I'm not going to recite a strategy in
this room — that's exactly what the PRD has to earn. What I'd ask leadership to hold: if the
blended number moves, or any cohort's number moves, we get a same-day explanation of why and
whether it's migration-related.

**Priya:** Define the rollback.

**Sam:** In the PRD. The honest answer today is "it depends on how far in we are" — and the plan
has to price that per phase: what stopping looks like, what changing course means for customers
already moved, and where soak time beats speed. Hand-waving any of those is how migrations become
incidents.

**Dana:** Timeline. I don't want bravado, I want a plan.

**Sam:** The PRD will carry the sequencing — what lands when, working backward from June.
Constraints you've set: Wave 2 complete and
stable before the June 1–15 renewal crunch — call it a third of our annuals renewing in a fortnight.
Kickoff is the tenth. The plan you get next week will be phased with named risks and owners.

**Priya:** I want the plan to include the decision I owe you on taxes. Ruth, where are we?

**Ruth:** No decision yet — deliberately. What I know: we sell into the EU and Canada, VAT and GST
rules change under us, and I have no interest in owning rate tables by hand forever. But we have a
handful of EU business accounts with VAT IDs and a few flagged exempt. Whatever we pick must handle
reverse-charge for those, and I need it working before the *first* Stripe invoice, not retrofitted.

**Priya:** So the PRD should state the tax decision and why. Fine — I'll ratify whatever it
recommends, but it needs to *recommend*.

**Dana:** Anything the PRD needs that we haven't funded?

**Sam:** Two things. One: support load. Descriptor change plus re-auth emails means a ticket bump
in week one of Wave 2 — Amara's asking for temp coverage and better macros, it's cheap, approve it.
Two: comms. Marketing wants every customer told before anything changes. That's sequencing inside
the batches, not a big cost.

**Dana:** Both approved contingent on the plan showing when. Last thing — the Founders cohort.
Eight customers on a 2005 plan at 4.99 forever. What happens to *them*?

**Priya:** Technically they're just subscriptions like anyone else — same engine, same token flow.
But there's a real question buried here: what *is* Founders after the move? Revenue-wise it's
pocket change; trust-wise it's everything. Those people predate the tier system. I want this one
decided deliberately — options on paper, a recommendation, a named decision-maker — not silently
decided by whoever writes the mapping file.

**Dana:** Agreed — put it in the PRD as an open decision with options and a recommendation. We'll
ratify it in the review. Okay: PRD next week. Sam — it needs to be buildable. I want engineers
starting from it, not translating it.

---

*Transcript ends. 31 minutes.*
