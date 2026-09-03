# Summary: Reporting / Data Pipeline Slack Thread

**Source:** `corpus/03_reporting_data_pipeline_thread.md` — Slack export, `#data-reporting`,
Mar 4–6 2025. Participants: Wren Tanaka (Head of Data), Dev Kapoor, Mike (Finance).

## Overview
The reporting team stakes its data requirements early because retrofitting is expensive ("one-way
door: can't reshape it after 100 subscriptions exist"). Confirms **Stripe Data Pipeline** as the
chosen ingestion mechanism (daily loads of charges/invoices/subscriptions/customers/credit
notes/refunds into the warehouse) and lists exactly which business dimensions must survive the move
since Stripe has no native concept of them.

## Stakeholder decisions
- **Wren Tanaka / Dev Kapoor:** Stripe Data Pipeline is the chosen warehouse ingestion mechanism
  (decided pre-thread, in January) — daily loads, all core objects.
- **Wren Tanaka:** Every Stripe object Streamhaven creates must be joinable back to `CUST-####` /
  `SUB-####` in the warehouse — called "non-negotiable," since Priya reconciles the first two
  months to the penny.
- **Mike (Finance):** Any statement descriptor change must be flagged to finance in advance — false
  fraud alerts hit revenue ops when descriptors change.
- **Mike (Finance):** Month-end numbers must tie out to the penny for the first two months
  post-cutover; "source of truth" during that window must be explicitly decided and documented.

## What must stay the same
- All report cuts that exist today must keep working post-migration: MRR by plan/interval/currency,
  legacy (Founders) vs. new-plan revenue split, acquisition-channel cohorts, discount/redemption
  tracking (including "how far into a discount window" per subscriber), churn/dunning recovery
  metrics, region/currency (USD/CAD/EUR, EU vs NA) rollups.
- The audit join back to internal `CUST-####`/`SUB-####` IDs for every Stripe object.

## What must change
- Business dimensions Stripe doesn't natively model must be carried some other way (in Stripe
  metadata, or joined purely from the warehouse side) — decision needed **before** Wave 2 creates
  100 live subscriptions, since it can't be reshaped afterward:
  - Founders/legacy plan tag.
  - Acquisition channel (organic/paid_search/referral/partner).
  - Discount window progress (e.g., START25's month-by-month wind-down).
  - Region/currency splits (already exist in data — must simply not be dropped).
- Churn/dunning-recovery reporting shifts to reading Stripe's failed-payment/recovery data instead
  of the in-house engine's equivalents — needs field-mapping decisions.
- Historical invoice backfill question: do pre-cutover invoices get recreated in Stripe, or does the
  warehouse keep serving old history while Stripe serves new? Flagged explicitly as unresolved —
  needed to avoid a visible "cliff" in revenue dashboards at cutover.

## Open questions raised (for PRD)
1. Which business dimensions travel *inside* Stripe (e.g., as metadata) vs. exist *only* in the
   warehouse, and how are they attached to each object at creation time?
2. What is the definitive "source of truth" for month-end financials during the overlap period
   (first two months post-cutover)?
3. Historical invoice backfill strategy — does dashboard history get recreated in Stripe or does the
   warehouse continue to serve pre-cutover data seamlessly across the join?
4. Statement descriptor decision needs an explicit heads-up to Finance before it ships (cross-ref
   with support/eng thread).
