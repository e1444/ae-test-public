# Summary: Leadership Checkpoint

**Source:** `corpus/05_leadership_checkpoint.md` — transcript, Mar 7 2025. Attendees: Dana Whitfield
(CEO), Priya Nair (CFO), Ruth Calloway (Controller), Sam Okafor (VP Eng).

## Overview
Executive status/risk review ahead of the PRD delivery. Presses engineering for *mechanisms* rather
than intentions on the two CFO red lines (no double-billing, renewal dates preserved), surfaces
authorization-rate risk as a board-level concern, demands an explicit rollback plan, and forces a
decision framework (not a decision) for the Founders cohort.

## Stakeholder decisions
- **Priya Nair:** Demands the PRD show double-billing prevention and renewal-date preservation as
  concrete *mechanisms*, not stated intentions.
- **Priya Nair (four auth-rate risks to address in the PRD):**
  1. Every migrated card is unproven at Stripe — first charge is the riskiest of the year.
  2. In-house dunning/retry knowledge of the 6 mid-cycle customers is lost when retry
     infrastructure moves to Stripe.
  3. Descriptor change + re-auth + new processor = elevated fraud-panic support load.
  4. (Ruth's addition) Reconciliation ambiguity if Stripe and the old engine disagree about a charge
     that happened *during* the cutover window.
- **Sam Okafor:** Commits that all four risks go in the PRD, auth rate first; asks leadership for a
  same-day root-cause commitment (migration-related or not) if the blended auth number or any
  cohort moves.
- **Sam Okafor:** Rollback plan must be phased — "it depends how far in we are" is treated as the
  honest starting framing, but the PRD must price rollback cost/impact per phase, not hand-wave it.
- **Dana Whitfield:** Approves two previously-unfunded asks contingent on the PRD showing timing:
  (1) temporary support coverage + better macros for the week-one ticket bump, (2) sequencing
  customer comms ahead of any customer-facing change.
- **Dana Whitfield / Priya Nair:** Founders cohort (8 customers, $4.99/mo since 2005) must be
  decided **deliberately** — options on paper, a recommendation, and a **named decision-maker** —
  explicitly not left to whoever happens to write the mapping file. Leadership will ratify in a
  future review, not decide today.
- **Ruth Calloway:** No tax decision yet, deliberately deferred — but whatever gets picked must
  handle EU business VAT-ID reverse-charge cases and existing tax-exempt flags, and must be working
  before the **first** Stripe invoice, not retrofitted. Priya will ratify whatever the PRD
  recommends, provided it actually recommends something (not just presents options).

## What must stay the same
- Renewal dates and no-double-billing guarantees — framed even more strongly as board-visible,
  mechanism-backed commitments.
- Auth rate (91.4% blended, 93% NA / 88% EU baseline) — any movement needs a same-day explanation.
- Three-currency operation, June-deadline sequencing (unchanged from earlier docs).

## What must change / new requirements introduced here
- PRD must include a phased rollback plan (cost/impact of stopping or reversing at each stage).
- PRD must include a same-day auth-rate-movement explanation process (operational commitment, not
  just a design).
- Temporary support coverage and improved macros need to be scheduled into the sequencing (funded,
  contingent on timing being shown).
- Founders cohort needs a dedicated decision framework in the PRD (options + recommendation + named
  decision-maker), to be ratified in a separate review — not shipped as a default/implicit choice.

## Open questions raised (for PRD)
1. Founders cohort treatment — options, a recommendation, and who signs off (explicitly not
   engineering's call to make silently).
2. Tax approach — Finance/Ruth still hasn't decided; PRD must state a recommendation Priya can
   ratify, working before the first Stripe invoice, handling EU VAT reverse-charge + exempt flags.
3. Rollback/rollback-cost plan per migration phase.
4. Process for detecting and explaining auth-rate movement same-day during the migration window.
