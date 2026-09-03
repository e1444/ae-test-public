# Summary: Project Relay Announcement

**Source:** `corpus/00_project_relay_announcement.md` — CEO all-staff email (Dana Whitfield, Feb 24
2025) plus CFO reply (Priya Nair).

## Overview
Kickoff announcement for the billing migration from Streamhaven's in-house engine to Stripe
("Project Relay"). Sets the two-wave structure, the hard deadline, and pulls in cross-functional
requirements before engineering starts designing. Priya's reply immediately attaches finance's
non-negotiables.

## Stakeholder decisions (final, not to be relitigated)
- **Dana Whitfield (CEO):** Not using Stripe's managed migration service — engineering builds its
  own scripts, for repeatability/control/in-house tooling.
- **Dana Whitfield:** Two waves — Wave 1 (customers + payment methods), Wave 2 (billing: plans,
  prices, coupons, subscriptions). Wave 2 cutover must finish before the June renewal wave.
- **Dana Whitfield:** Operating currencies stay USD/CAD/EUR only — no new markets mid-project.
- **Dana Whitfield:** Customer-facing surface changes only where unavoidable; Support + Marketing
  own the comms plan.
- **Priya Nair (CFO):** Zero tolerance for double-billing — one charge per customer per period.
- **Priya Nair:** Every migrated subscription must preserve renewal date, price, and discount
  state (revenue recognition depends on it; she reconciles the first two months to the penny).
- **Priya Nair:** Tax approach is explicitly deferred to Wave 2 planning, but must be *decided*
  before the first Stripe invoice is created.
- **Priya Nair:** Auth rate (91.4% blended last quarter) is a board-level metric — wants a written
  plan for protecting it through the transition.

## What must stay the same
- Renewal dates, prices, discount/coupon state for every migrated subscription.
- Billing currencies (USD/CAD/EUR) — no expansion.
- One charge per customer per billing period (no double-billing, ever).
- Customer-facing experience, except where the migration technically requires a change.

## What must change
- Underlying billing system of record moves from the in-house engine to Stripe.
- Card data moves from AtlasPay-tokenized storage into Stripe's vault (Wave 1).
- Plans/prices/coupons/subscriptions get recreated as native Stripe objects (Wave 2).
- Tax handling approach — currently undecided, must be resolved before go-live invoicing.

## Open items flagged here for later resolution
- Tax approach (owner: Priya Nair / Finance).
- Auth-rate protection plan (owner: Engineering, per Priya's request).
