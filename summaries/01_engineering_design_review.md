# Summary: Engineering Design Review

**Source:** `corpus/01_engineering_design_review.md` — transcript, Mar 5 2025. Attendees: Sam Okafor
(VP Eng), Jonas Meyer (Staff Eng, Billing), Elena Sokolov (Sr Backend Eng), Wren Tanaka (Head of
Data), Amara Diallo (Head of Support, partial).

## Overview
The core architecture walkthrough for Relay. Covers the three internal tables that matter
(`customers`, `subscriptions`, `payment_methods`), the Wave 1 card-import mechanics via Stripe's
SFTP-based import, and enumerates the hard Wave 2 problems (dunning, scheduled downgrades,
discounts, STREAM30 monthly-only enforcement, trials, renewal-date integrity, the cutover/state
machine, and bidirectional data sync). This is the meeting that generates most of the PRD's open
design questions.

## Stakeholder decisions
- **Sam Okafor:** Build the migration as if scale were 100,000 customers, not 100 — idempotent,
  resumable, observable, even though today's count is ~100.
- **Sam Okafor:** The in-house engine (including its dunning engine) is being fully retired — no
  "old engine lives on for a subclass of customers." Whatever the dunning decision is, it must
  resolve to something running on Stripe's side.
- **Sam Okafor:** Renewal dates must be preserved — stated as a hard requirement, not a preference.
- **Sam Okafor:** The PRD must define an explicit state machine for "who bills where" (per-
  subscription), queryable, not a boolean/memory-based answer.
- **Sam Okafor:** PRD must address how Stripe-side events (renewals, failures, cancellations) flow
  back into Streamhaven's systems for every consumer (internal tables, warehouse, finance) — not
  a single-consumer design.
- **Elena Sokolov (proposed, unchallenged):** Wave 1 mechanics — AtlasPay vault export → PGP-
  encrypt → SFTP to Stripe migration inbox with manifest → Stripe creates Customer + PaymentMethod
  per row → response file returns old→new ID mapping only, cards only (no bank/wallet methods in
  scope).
- Explicitly deferred to the PRD as **named open decisions with owners**, not decided here: tax
  approach (Priya), dunning policy for the 6 mid-cycle customers, STREAM30 implementation
  (Marketing owns outcome, Engineering owns non-breakage), exact statement descriptor string.

## What must stay the same
- Three-currency operation (USD/CAD/EUR).
- Renewal dates, prices, and discount state per subscription (repeated from announcement).
- Wave separation: Wave 1 (cards) must not touch customer-facing billing at all — no charging
  through Stripe until Wave 2 cutover.

## What must change
- Migration engineering must be built for scale (idempotent/resumable/observable), not "quick
  script for 100 rows."
- Card data (PANs) must leave Streamhaven/AtlasPay via encrypted SFTP import into Stripe — one-time
  operation per card.
- New Stripe IDs (`cus_`, `pm_`, eventually `sub_`) need a home in Streamhaven's own tables, traceable
  back to `CUST-####`/`SUB-####` in both directions.
- Streamhaven's tables need new fields/states to represent "is this customer live on Stripe
  tonight?" — currently no such field exists.
- Dunning: in-house 30-day/4-retry engine is retired; 6 customers currently mid-cycle need an
  explicit decision (not yet made) for how their in-flight retry state is handled.
- Scheduled downgrades: `pending_plan_change`/`pending_plan_effective_at` (5 customers, all
  pro→basic effective April) have no defined Stripe-side representation yet — must be solved before
  cutover or customers get billed the wrong plan.
- Discounts: 5 coupons in flight; `discounted_periods_remaining` tracks what's "owed" — carry-over
  mechanism is undecided (flagged as the failure mode that costs customers silently, discovered at
  renewal).
- STREAM30: monthly-only restriction is enforced by the old engine's redemption logic, not a Stripe
  primitive — engineering's early read is this doesn't transfer automatically to Stripe coupons;
  needs verification, not assumption, plus a decided enforcement mechanism.
- Self-serve payment flows (add/replace/remove card, cancel) currently hit the in-house engine;
  need new implementations once Stripe is system of record.

## Open questions raised (for PRD to resolve, with named owners)
1. How are Stripe object IDs cross-referenced back to `CUST-####`/`SUB-####` (bidirectional keying)?
2. What does "live on Stripe" mean operationally — explicit state machine needed.
3. Dunning treatment for the 6 mid-cycle customers.
4. Scheduled-downgrade representation in Stripe for the 5 queued pro→basic customers.
5. Discount/coupon carry-over mechanism (especially partially-consumed windows like START25).
6. STREAM30 monthly-only enforcement post-migration (verify assumption; design guardrail).
7. Statement descriptor decision (options, not final string).
8. Self-serve card/cancel flow redesign.
9. Stripe→Streamhaven data sync mechanism (webhooks or otherwise) for renewals/failures/cancels.

## Key numbers surfaced
- ~100 customers/subscriptions/payment methods.
- Wave 1 test batch: 15 tokens submitted, 12 returned (3 silent failures: 2 expired cards, 1
  bank-closed account).
- 6 customers mid-dunning cycle right now.
- 5 customers with queued pro→basic downgrades, effective April 2025.
- 5 live coupons (detailed in marketing review), several with partially-consumed windows.
- 4 customers mid-trial.
- 12 price objects needed (2 plans × 2 intervals × 3 currencies) + 1 Founders price.
