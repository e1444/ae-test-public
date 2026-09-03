# Summary: Marketing Coupon Review

**Source:** `corpus/04_marketing_coupon_review.md` — transcript, Mar 6 2025. Attendees: Chloe
Baptiste (VP Marketing), Luis Herrera (Growth), Jonas Meyer (Eng), Elena Sokolov (Eng).

## Overview
Full inventory and exact mechanics of the five live coupons, surfaced so engineering can design
carry-over before Wave 2. Surfaces two genuinely unresolved product questions (STREAM30's
monthly-only enforcement, and TAKE5's fixed-amount cross-currency meaning) and locks a comms
requirement around promo continuity and the Founders cohort.

## Stakeholder decisions
- **Chloe Baptiste:** "Site is truth" for coupon copy — a stale blog post claiming "3 months of 20%
  off" for FRIEND50 is confirmed wrong and should be ignored.
- **Chloe Baptiste (ratified pending PRD):** TAKE5 ($5 off, one-shot) should convert to local-
  currency equivalents (5 CAD / 5 EUR-equivalent) rather than a flat USD charge — Luis's
  recommendation, Chloe ratifies, final wording lands in the PRD.
- **Chloe Baptiste:** Every migrating customer must hear about the change **from Streamhaven
  first** — no descriptor surprises, no confusing failed-card emails. Comms plan is a named PRD
  section.
- **Chloe Baptiste:** Founders cohort gets deliberately gentler, personal-toned communications —
  they're the longest-tenured customers and "will call."
- **Priya Nair (referenced):** Renewal dates preserved is already a written commitment from Priya —
  Chloe asks for it to appear in the PRD as a hard requirement with an explicit test, since one
  reset kills the whole promo program's trust for a year.
- **Jonas Meyer:** STREAM30's monthly-only rule is enforced by Streamhaven's engine at redemption
  time, not a portable rule — his early read is it will **not** carry over into Stripe coupon
  config automatically. Explicitly flagged as "verify, don't assume."

## Coupon inventory (exact mechanics — verified counts against data)
| Code | Mechanic | Window | Active subs |
|---|---|---|---|
| `START25` | 10% off | first 5 **consecutive monthly** invoices from signup | 6 (partway through) |
| `FRIEND50` | 50% off | first 3 monthly invoices (referral promo) | 4 |
| `STREAM30` | 30% off | monthly billing **only** — never applied to annual | 5 |
| `TAKE5` | $5 off | single invoice, one-shot | 2 |
| `ANNUAL20` | 20% off | forever, until cancelled (annual-signup promo) | 3 |

## What must stay the same
- Every partially-consumed discount window's remaining balance ("owed to them") must carry over
  exactly — tracked today via `discounted_periods_remaining`. Losing track of this is called out as
  "the failure mode that fails quietly" (surfaces at renewal, months later).
- Renewal dates must not shift for any customer — framed as a hard requirement with a required test,
  not just a design goal.
- STREAM30 should ideally stay live/redeemable for *new* signups through cutover — Luis prefers
  uglier enforcement over pausing a live, evergreen promo.

## What must change
- STREAM30 needs a **new enforcement mechanism** on the Stripe side (Stripe coupons have no native
  "monthly-only" restriction) — design owed by engineering, decision on approach owned by marketing/
  engineering jointly.
- TAKE5 needs an explicit multi-currency definition instead of a single implicit USD amount (become
  local-currency equivalents per Chloe's ratification).
- Comms: a dedicated notification plan/matrix, with a distinctly gentler dedicated track for
  Founders customers.

## Open questions raised (for PRD, with a recommendation already leaning)
1. **STREAM30 monthly-only enforcement post-migration** — verify Stripe coupon capabilities; design
   a guardrail (e.g., product/price-scoped coupon restriction or subscription-level validation) so
   it can't accidentally apply to annual plans. Also decide continuity strategy for cutover window
   (recommendation: keep it live, accept messier enforcement, rather than pause a live promo).
2. **TAKE5 cross-currency amount** — leaning decision: local-currency equivalents (5 CAD/5 EUR), to
   be stated explicitly in the PRD.
3. Comms plan structure — full notification matrix (who/what/when), Founders given a dedicated,
   handwritten-feeling track.
