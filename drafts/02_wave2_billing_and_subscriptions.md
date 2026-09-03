# Draft: Pillar 3 — Wave 2: Billing & Subscription Migration

**Status:** Working draft for `PRD.md`. Follows the same process as
`drafts/01_launch_strategy_and_wave1.md`: design decisions grounded in the corpus + verified data,
explicit data model changes, sample Stripe API calls, an engineering-owned open-decisions table, and
a separate leadership/Stripe-verification ambiguities table.

**Inputs used:** `summaries/01`, `02`, `04` (design review, FAQ, coupon review), plus verified counts
from `data/subscriptions.csv`: 8 Founders, 6 mid-dunning (retry_1×1, retry_2×2, retry_3×1,
final_notice×2), 5 queued pro→basic downgrades (all effective April 2025), 3 customer-initiated
cancellations queued for period end (`cancel_at_period_end=true`; distinct from the downgrade
cohort — no coupon or pending-change overlap), 4 trialing, 5 coupons (START25×6, FRIEND50×4,
STREAM30×5, TAKE5×2, ANNUAL20×3), 4 business accounts with a VAT ID and 3 flagged tax-exempt
(not fully overlapping sets), and a full price-book cross-check (below).

**Depends on:** Wave 1 must be complete for a subscription's customer (Stripe `cus_…`/`pm_…` exist)
before Wave 2 can create a Stripe subscription for them. Uses the `billing_system` state machine
and batching plan from the launch-strategy draft unchanged — this document only covers what gets
*built* for each subscription, not when each batch runs.

**Price-book integrity check (done before writing this draft):** every one of the 100 subscriptions'
`price_amount_cents` matches the documented price book exactly for its plan/interval/currency — no
customer is on a non-standard negotiated price. This matters because it means every subscription can
be pointed at one of a small, static set of shared Stripe Price objects; nobody needs an individually
custom Price. If this check had failed for even one row, that row would need its own Price object
instead.

**Verified** — the full plan/interval/currency/price cross-tab from `data/subscriptions.csv`, with
no row outside the documented price book:

| Plan | Interval | Currency | `price_amount_cents` | Subscriptions |
|---|---|---|---|---|
| basic | monthly | USD | 499 | 24 |
| basic | monthly | CAD | 699 | 9 |
| basic | monthly | EUR | 499 | 7 |
| basic | annual | USD | 4990 | 12 |
| basic | annual | CAD | 6990 | 2 |
| basic | annual | EUR | 4990 | 3 |
| pro | monthly | USD | 2999 | 16 |
| pro | monthly | CAD | 3999 | 6 |
| pro | monthly | EUR | 2999 | 5 |
| pro | annual | USD | 29990 | 3 |
| pro | annual | CAD | 39990 | 3 |
| pro | annual | EUR | 29990 | 2 |
| founders | monthly | USD | 499 | 8 |

13 distinct (plan, interval, currency) combinations — one per Price object in the catalog below,
confirming the 13-Price count is exactly right with no leftover or missing combination.

---

## Guiding principles
1. **Prices are shared, static, and created once.** Subscriptions reference them; they are never
   created per-customer.
2. **A coupon's Stripe representation must reproduce what's *owed*, not what's advertised.** A
   customer three months into a five-month promo does not get a fresh five months at migration —
   the "owed to them" framing from `summaries/04` is the literal design spec.
3. **Every Stripe primitive chosen here is picked because it's the correct native mechanism, not
   because it's the easiest to bolt on.** Where Stripe has no native answer (STREAM30's monthly-only
   rule), that gap is named explicitly rather than quietly left to application code alone.
4. **Nothing in Wave 2 is invented file-by-file at migration time.** The catalog (products, prices,
   coupon variants) is designed and created *before* any customer migration begins, then every
   subscription-creation call only ever *references* existing IDs.

---

## Design decisions

### 1. Product & price catalog
Rather than one "Basic" and one "Pro" Product with mixed-interval Prices underneath, **create a
separate Product per plan × billing interval** (`Basic Monthly`, `Basic Annual`, `Pro Monthly`,
`Pro Annual`, `Founders`). This is a deliberate, non-obvious catalog choice, made for two concrete
reasons surfaced in the corpus:

- **STREAM30 must be structurally incapable of applying to annual plans** (Chloe's hard requirement).
  Stripe Coupons support an `applies_to.products` restriction — but only at the *Product* level. If
  monthly and annual shared one "Basic" Product, that restriction couldn't distinguish them. Splitting
  the catalog by interval turns "monthly-only" into a platform-enforced guarantee instead of an
  application-layer promise, which is the "verified, not assumed" bar Jonas asked for.
- **Reporting's legacy-vs-new and plan/interval MRR cuts** (`summaries/03`) map directly onto Product
  IDs, which flow into Stripe Data Pipeline's warehouse tables for free — no metadata-string parsing
  required downstream.

Thirteen Price objects total (matches Jonas's "twelve plus Founders"): 2 plans × 2 intervals × 3
currencies (12) + 1 Founders (USD monthly only, price-locked).

Each Price carries `metadata.legacy_price_amount_cents` and `metadata.plan_interval_key` (e.g.
`pro_monthly_usd`) so the migration job can look up the correct Price by a deterministic key instead
of a fragile amount-based match, and so a Stripe-side audit can always trace a Price back to the
price-book row it represents.

**This is also, technically, the Founders answer — but it must not stay silent.** A dedicated,
isolated `Founders` Product/Price (closed to new signups, price permanently locked at $4.99) is what
makes it structurally impossible for a downgrade path, a bulk price update, or a support macro to
ever move a Founders customer onto Basic pricing or vice versa. That satisfies Dana/Priya's
requirement mechanically, but Priya was explicit that Founders needs to be **decided deliberately —
options on paper, a recommendation, and a named decision-maker — not silently decided by whoever
writes the mapping file.** This draft is exactly that silent author if this design isn't also carried
into the Open Decisions table below as its own ratified line item.

### 2. Coupon carry-over mechanism
The core problem (per `summaries/04`, Jonas): Stripe coupon objects are shared and centrally define
their own duration — attaching the live "public" `START25` coupon (duration_in_months: 5) to a
migrated subscriber would hand them 5 *new* months instead of however many they have *left*. The
public coupon must keep working for new signups unchanged, so migrated customers need distinct
coupon objects, not the same one.

**Mechanism:** for every repeating coupon, create one **migration-variant coupon per distinct
remaining-period count actually present in the data**, rather than one per customer (small, fixed
set, created once):

| Public coupon | Public duration | Remaining-period values in data (subscriber count) | Migration-variant coupons to create |
|---|---|---|---|
| `START25` (10% off) | 5 months | 2 (×3), 3 (×1), 4 (×2) | `START25_MIG_2MO`, `START25_MIG_3MO`, `START25_MIG_4MO` |
| `FRIEND50` (50% off) | 3 months | 2 (×1), 3 (×3) | `FRIEND50_MIG_2MO`, `FRIEND50_MIG_3MO` |
| `STREAM30` (30% off) | forever | n/a (not period-limited) | none — public coupon applies directly, restricted via `applies_to.products` to the two monthly Products |
| `TAKE5` ($5 off, one-shot) | once | n/a (2 subscribers) | per-currency variants only (see below), not per-period |
| `ANNUAL20` (20% off) | forever | n/a (3 subscribers) | applied directly per subscription (see below) — not re-exposed as a redeemable code |

Per-variant subscriber counts (verified against `data/subscriptions.csv`) size both the one-time
catalog-creation job and the pre-flight coupon-match audit's expected fan-out: 6 subscriptions need a
`START25_MIG_*` variant, 4 need a `FRIEND50_MIG_*` variant, 5 attach the plain `STREAM30` coupon, 2
need a `TAKE5_*` currency variant, and 3 get the direct `ANNUAL20` subscription-level discount — 20
discounted subscriptions total, matching the coupon inventory from `summaries/04`.

Each migrated subscription is given the migration-variant coupon matching its *exact*
`discounted_periods_remaining` value at migration time — not the public coupon. A pre-migration
audit script diffs each subscription's expected discount (percent + remaining periods, computed from
today's `subscriptions.csv` row) against what the Stripe subscription actually has attached, and
**blocks that subscription from reaching `stripe_live`** on any mismatch. This is the concrete
mechanism for "not a single lost discount," not a hope.

**TAKE5** (per Chloe's ratification): three currency-specific coupons, each a flat local-currency
five, not an FX conversion — `TAKE5_USD` (`amount_off: 500, currency: usd`), `TAKE5_CAD`
(`amount_off: 500, currency: cad`), `TAKE5_EUR` (`amount_off: 500, currency: eur`) — each
`duration: once`. Stripe fixed-amount coupons are inherently currency-scoped, so this isn't extra
design, just correctly using the primitive.

**ANNUAL20**: per the transcript, this was a *time-boxed* signup promo ("last year's push") — it is
not a currently-redeemable public offer, only a standing discount on the 3 existing subscriptions
that already have it. Recommendation: do **not** expose it as a redeemable Promotion Code at all;
instead create one private Coupon (`duration: forever`, `percent_off: 20`, no `Promotion Code`
attached to it) and attach it directly to just those 3 subscriptions via the subscription's
`discounts` array. To be precise about the mechanics: Stripe has no "raw percent-off, no coupon
object" primitive — a Coupon resource is still created either way; "not a redeemable code" means it
never gets a Promotion Code wrapper a customer or agent could type in, not that no Coupon object
exists. This avoids accidentally reviving a retired promo as something a support agent or script
could redeem again.

### 3. STREAM30 continuity through cutover
Per Luis's stated preference ("I'd rather it keep working — even if the enforcement is uglier — than
pull a live promo"): STREAM30 stays active for new signups throughout the migration window. Because
its Stripe representation is the Product-scoped coupon from #1/#2 above, this requires no special
handling during cutover — it's simply live in Stripe from the moment the monthly Products/coupon
exist, independent of any individual customer's migration batch.

### 4. Scheduled downgrades → Stripe Subscription Schedules
The correct native primitive for "queued plan change effective at period end" is a **Subscription
Schedule** with two phases, not a manual cron job that swaps the subscription's price later:
- Phase 1: current price (Pro), `end_date` = the customer's exact `pending_plan_effective_at`.
- Phase 2: target price (Basic), starting at that same boundary, `proration_behavior: none` (it's a
  scheduled change at a period boundary, not a mid-cycle upgrade/downgrade — no proration is owed
  either direction).

This is a Stripe-native, queryable object — closing the exact gap Jonas flagged ("nobody has worked
out what this looks like on Stripe's side"). All 5 currently-queued `pro_to_basic` customers get a
schedule created at migration time; the schedule, not a Streamhaven cron job, is what actually
executes the downgrade on the effective date.

### 5. Trials
4 customers are currently mid-trial. Their Stripe subscriptions are created with `trial_end` set to
the *exact* legacy `trial_end` timestamp — a straight one-to-one field mapping, no design gap here.

### 6. Renewal-date preservation mechanism
This reuses the same Stripe field (`trial_end`) for a different purpose, which needs to be called out
explicitly so it isn't confused with #5: for a **non-trialing** subscription, Stripe has no direct
"create this subscription now but don't invoice until date X, with no proration" parameter other than
`trial_end`. The migration therefore creates every non-trialing subscription with `trial_end` set to
that customer's exact `current_period_end` — Stripe will not invoice until that instant, then
auto-transitions and bills normally from there on the customer's real cadence. This is a deliberate,
recommended-by-Stripe reuse of the trial mechanism as a billing-date anchor, and it is the literal
mechanism behind "renewal dates preserved" that Priya asked to see, not an intention.

**Reporting side-effect this creates (flagged, not silently absorbed):** the Stripe subscription's
`status` will legitimately read `trialing` for every migrated customer until their first real Stripe
invoice, indistinguishable from a real trial unless tagged. Every subscription created this way
carries `metadata.migration_billing_anchor: "true"` (distinct from real trials, which do not) so
Wren's warehouse join can correctly exclude anchor "trials" from any trial/conversion-funnel metric.

### 7. Dunning treatment for the 6 mid-cycle customers
Per Sam's explicit ruling, the old engine is retired outright — "the old engine lives on for a
subclass of customers" was rejected as an answer. Given the FAQ's unresolved-and-unverified premise
(Stripe likely cannot import "retry 3 of 4"), the recommended design **does not attempt to replicate
in-house retry progress inside Stripe at all**:
- The specific overdue invoice these 6 customers currently owe is treated as a **legacy
  accounts-receivable matter**, resolved out-of-band (one bounded final legacy collection attempt, or
  written off per Finance policy) — it is not carried into Stripe as a Stripe-side invoice.
- Once resolved (paid or written off), the customer's subscription migrates like any other, starting
  a **clean forward-dated cycle** in Stripe with Stripe's own (Smart Retries) dunning active from
  their next real payment attempt onward.
- This means the retry *count* legitimately resets — treated here as a conscious, named tradeoff (see
  ambiguity #4 below) rather than a silent behavior change, since nobody has ratified it yet.
- Per Amara's pairing requirement: whichever team owns this decision also owns turning off the legacy
  dunning emails for these 6 specific customers the moment they're marked for migration, so nobody
  gets a legacy retry email and a Stripe payment-failure email in the same week.

### 8. Tax handling (recommendation — this doc is where it must land, not stay open)
The FAQ and leadership checkpoint both mark tax as open, but also both require the PRD to actually
**recommend** something Priya can ratify before this doc's own subscription-creation calls generate
the first Stripe invoice — leaving tax unaddressed here contradicts the doc's own worked examples.
**Recommendation: Stripe Tax, not a ported in-house rate table.** Concretely:
- Every subscription created here sets `automatic_tax: {"enabled": true}` — Stripe calculates and
  applies the correct rate per customer address/tax status at invoice time, which is precisely the
  "stop owning rate tables by hand" outcome Ruth asked for.
- For the 4 business customers with a VAT ID (`customers.tax_id`), attach a `tax_id` object
  (`type: eu_vat`) to their Stripe Customer during Wave 1/enrichment; Stripe Tax then applies
  reverse-charge treatment automatically for EU B2B sales instead of us hand-coding it.
- For the 3 accounts flagged `tax_exempt = true`, set `customer.tax_exempt` to Stripe's matching
  value (`"exempt"` for a straightforward exemption, `"reverse"` for a VAT-ID business account that's
  exempt via reverse charge specifically — these are not the same accounts, so each of the 4 VAT-ID
  customers and 3 exempt-flagged customers needs its own correct value, not one blanket setting).
- This is a **recommendation for Priya/Ruth to ratify**, not a unilateral engineering decision — see
  Open Decisions below — but it is a real recommendation, not another open flag.

### 9. Preserving in-flight cancellation intent
Three currently-**active** subscriptions already have `cancel_at_period_end = true`
(`SUB-1020`, `SUB-1064`, `SUB-1067`) — these customers already asked to cancel and are riding out a
paid period they've already committed to. The `trial_end`-anchor mechanism in decision #6 says
nothing about this flag; migrated naively, these 3 people would come out the other side as an
ordinary auto-renewing Stripe subscription and get billed again after explicitly canceling — a
different-but-equally-real breach of "nobody's charged twice" as leadership defined it. **Fix:**
`subscriptions.cancel_at_period_end` is copied 1:1 onto the created Stripe subscription's
`cancel_at_period_end` field at creation time, and the pre-flight audit (already required for
coupons and schedules) additionally asserts this flag matches for these 3 rows before `stripe_live`.

### Double-charge / lost-discount prevention (ties to the launch-strategy draft)
This document doesn't re-derive the `billing_system` state-machine guard from
`drafts/01_launch_strategy_and_wave1.md` — it relies on it unchanged. What Wave 2 adds on top:
- The **coupon-match audit** (design decision #2) as a discount-specific pre-flight gate.
- The **Subscription Schedule** itself as the downgrade-correctness gate — a schedule either exists
  with the right phases or it doesn't; there's no in-between state where a downgrade could be
  silently dropped.
- The existing pre-flight invoice-preview check (from the launch doc) additionally validates that
  the *discounted* amount matches expectations, not just the raw price — catching a missing or
  wrong-variant coupon before `stripe_live` is reached.
- The same pre-flight check also verifies `cancel_at_period_end` parity (#9) — a customer who
  already canceled cannot be silently re-subscribed, and a still-active customer cannot be silently
  marked to cancel.

---

## Data model changes

```sql
-- static reference data: the Wave 2 catalog, created once, referenced by every migration
CREATE TABLE stripe_price_catalog (
  plan_interval_key   TEXT PRIMARY KEY,  -- e.g. 'pro_monthly_usd', 'founders_monthly_usd'
  stripe_product_id   TEXT NOT NULL,
  stripe_price_id     TEXT NOT NULL,
  legacy_price_amount_cents INT NOT NULL
);

-- static reference data: migration-variant coupons, created once per distinct remaining-period value
CREATE TABLE stripe_coupon_catalog (
  variant_key         TEXT PRIMARY KEY,  -- e.g. 'START25_MIG_3MO'
  base_coupon_code    TEXT NOT NULL,     -- 'START25'
  stripe_coupon_id    TEXT NOT NULL,
  remaining_periods    INT
);

-- per-subscription record of what was actually attached in Stripe, for the pre-flight audit
CREATE TABLE stripe_subscription_migrations (
  subscription_id       TEXT PRIMARY KEY REFERENCES subscriptions(subscription_id),
  stripe_subscription_id TEXT NOT NULL,
  stripe_price_id        TEXT NOT NULL,
  stripe_coupon_id       TEXT,             -- null if no discount
  stripe_schedule_id     TEXT,             -- null unless a downgrade is queued
  expected_amount_cents  INT NOT NULL,     -- computed pre-migration, post-discount
  expected_cancel_at_period_end BOOLEAN NOT NULL DEFAULT false,  -- from legacy subscriptions row
  audit_status           TEXT NOT NULL CHECK (audit_status IN ('pending','passed','failed')),
  migrated_at            TIMESTAMPTZ,
  created_at             TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

(`subscriptions.stripe_subscription_id` and `subscriptions.billing_system` are already added in the
Wave 1 draft — reused here, not redefined.)

---

## Sample Stripe API calls

**Price creation (one of thirteen, run once against the static catalog — Pro Monthly USD):**

```json
POST /v1/products
{
  "name": "Pro Monthly",
  "metadata": { "plan": "pro", "billing_interval": "monthly" }
}
```

```json
POST /v1/prices
{
  "product": "prod_<pro_monthly>",
  "currency": "usd",
  "unit_amount": 2999,
  "recurring": { "interval": "month" },
  "metadata": {
    "plan_interval_key": "pro_monthly_usd",
    "legacy_price_amount_cents": "2999"
  }
}
```
*`metadata.legacy_price_amount_cents` is the non-obvious field — it lets the pre-flight audit assert
"this Price's amount still matches the price book" without hardcoding numbers into migration code.*

**STREAM30 coupon, restricted to monthly Products only:**

```json
POST /v1/coupons
{
  "percent_off": 30,
  "duration": "forever",
  "name": "STREAM30",
  "applies_to": { "products": ["prod_<basic_monthly>", "prod_<pro_monthly>"] },
  "metadata": { "streamhaven_coupon_code": "STREAM30" }
}
```
*`applies_to.products` is the non-obvious field — this is the actual platform-enforced guarantee
that STREAM30 can never apply to an annual subscription, even from the Stripe Dashboard, which is
what Chloe explicitly asked for ("I need the guarantee").*

**START25 migration variant (3 months remaining), for a specific migrated subscriber:**

```json
POST /v1/coupons
{
  "percent_off": 10,
  "duration": "repeating",
  "duration_in_months": 3,
  "name": "START25 (migrated, 3 months remaining)",
  "metadata": {
    "streamhaven_coupon_code": "START25",
    "migration_variant": "true",
    "remaining_periods_at_migration": "3"
  }
}
```

**TAKE5, one of three currency-scoped coupons (CAD shown):**

```json
POST /v1/coupons
{
  "amount_off": 500,
  "currency": "cad",
  "duration": "once",
  "name": "TAKE5 (CAD)",
  "metadata": { "streamhaven_coupon_code": "TAKE5" }
}
```
*`currency` is the non-obvious field — Stripe requires a fixed-amount coupon to declare one currency;
this is what makes "5 CAD off" a flat local amount rather than a USD amount converted at charge time,
per Chloe's ratification.*

**ANNUAL20, private discount (no Promotion Code) attached directly to one of the 3 subscriptions:**

```json
POST /v1/coupons
{
  "percent_off": 20,
  "duration": "forever",
  "name": "ANNUAL20 (migrated, closed promo)",
  "metadata": { "streamhaven_coupon_code": "ANNUAL20", "promotion_code": "false" }
}
```
```json
POST /v1/subscriptions/sub_<existing>
{
  "discounts": [{ "coupon": "co_<annual20_coupon_above>" }]
}
```
*No `promotion_code` object is ever created for this coupon — that's the entire mechanism that keeps
a retired promo from being re-redeemable by anyone.*

**Subscription Schedule for a queued pro→basic downgrade (e.g. `SUB-1010`, effective
2025-04-09):**

```json
POST /v1/subscription_schedules
{
  "customer": "cus_<mapped_from_wave1>",
  "start_date": "now",
  "phases": [
    {
      "items": [{ "price": "price_<pro_monthly_cad>" }],
      "end_date": 1744190160
    },
    {
      "items": [{ "price": "price_<basic_monthly_cad>" }],
      "proration_behavior": "none"
    }
  ],
  "metadata": { "streamhaven_subscription_id": "SUB-1010" }
}
```
*`proration_behavior: "none"` on the second phase is the non-obvious field — without it, Stripe's
default behavior could generate a proration credit/charge at the boundary, which is exactly the kind
of unintended charge Priya's zero-double-billing rule is meant to catch.*

**Subscription creation using the renewal-date-preservation anchor (non-trialing customer,
`SUB-1023`, Pro Monthly USD, `current_period_end` = 2025-03-21T20:43:00Z):**

```json
POST /v1/subscriptions
{
  "customer": "cus_<mapped_from_wave1>",
  "items": [{ "price": "price_<pro_monthly_usd>" }],
  "trial_end": 1742589780,
  "proration_behavior": "none",
  "metadata": {
    "streamhaven_subscription_id": "SUB-1023",
    "migration_billing_anchor": "true"
  }
}
```
*`trial_end` here is the renewal-date-preservation mechanism (design decision #6), not a real trial —
`metadata.migration_billing_anchor` is what lets reporting and support tell the difference from the 4
genuine trialing customers.*

**The same call for one of the 3 already-canceling customers (`SUB-1020`, design decision #9) adds
one field:**

```json
POST /v1/subscriptions
{
  "customer": "cus_<mapped_from_wave1>",
  "items": [{ "price": "price_<pro_annual_usd>" }],
  "trial_end": 1745419560,
  "proration_behavior": "none",
  "cancel_at_period_end": true,
  "metadata": {
    "streamhaven_subscription_id": "SUB-1020",
    "migration_billing_anchor": "true"
  }
}
```
*`cancel_at_period_end: true` is copied straight from the legacy row — without it, this customer's
Stripe subscription would auto-renew despite having already asked to cancel.*

---

## Open decisions (Pillar 3 — engineering-level)

| # | Decision | Options | Recommendation | Owner |
|---|---|---|---|---|
| 1 | Catalog shape (Product-per-interval vs. Product-per-plan with Price-level metadata only) | Split by interval (this draft) vs. 2 Products with 6 Prices each | Split by interval — the only way to get a platform-enforced STREAM30 guarantee | Jonas Meyer |
| 2 | How to generate migration-variant coupons | Pre-create all plausible durations (1–12 months) speculatively vs. create only the durations actually observed in the data (this draft) | Only observed durations (2,3,4 for START25; 2,3 for FRIEND50) — smaller, auditable catalog; add a variant on demand if a new duration ever appears | Jonas Meyer |
| 3 | ANNUAL20 representation | Redeemable Promotion Code (reusable) vs. direct per-subscription discount, no code (this draft) | Direct per-subscription discount — matches its actual "closed promo" status | Chloe Baptiste (ratify) |
| 4 | **What is Founders after the move** (leadership-flagged, explicitly not engineering's call to make silently) | (a) Permanently isolated `Founders` Product/Price, closed cohort, price locked forever [structurally what this draft already builds] vs. (b) eventually consolidate onto Basic pricing with an equivalent permanent per-customer discount | (a) — it's already the technical design in decision #1, costs nothing extra, and preserves exactly the "since the dial-up era" trust Amara/Chloe flagged; (b) adds risk for a plan that's "pocket change" revenue-wise | Dana Whitfield (named per leadership checkpoint; Priya to ratify) |
| 5 | Resolution path for the 6 mid-dunning customers' currently-overdue invoice | One bounded final legacy-engine collection attempt before write-off vs. immediate write-off per Finance policy vs. manual white-glove outreach given it's only 6 people | One bounded final attempt (e.g. 7 days) via the legacy engine before it's retired, then write off per Finance policy if still unpaid — avoids abandoning collectible revenue while still hitting a hard retirement date | Ruth Calloway (process) / Amara Diallo (any customer-facing touch) |
| 6 | Tax engine | Stripe Tax (automatic, this draft's recommendation) vs. third-party (Avalara/TaxJar) integration vs. port the in-house rate tables as-is | Stripe Tax — matches Ruth's stated wish to stop owning rate tables by hand, and is natively wired into the same subscription-creation call this draft already designs | Priya Nair / Ruth Calloway (ratify per leadership checkpoint's explicit ask for a recommendation) |

## Ambiguities requiring leadership/Stripe clarification (Pillar 3)

| # | Ambiguity | Why it matters | Assumption if we proceed without an answer | Needs sign-off from |
|---|---|---|---|---|
| 1 | Whether Stripe's `applies_to.products` coupon restriction is actually available on Streamhaven's Stripe account/API version, and behaves as documented when a subscription mixes a restricted and unrestricted item | This is the entire mechanism claimed to make STREAM30 "structurally" monthly-only — if it doesn't work as assumed, the guarantee Chloe asked for doesn't exist and app-layer validation becomes load-bearing instead of a backstop | **We assume** `applies_to.products` behaves as documented and is available on the account; must be confirmed against Stripe's current API/account capabilities before this is presented as a solved guarantee | Elena Sokolov / Jonas Meyer |
| 2 | Whether reusing `trial_end` as a pure billing-date anchor (design decision #6) has any side effect Stripe treats differently from a "real" trial — e.g., whether Smart Retries, SCA/3DS behavior, or Stripe Tax calculation differs for a subscription that is technically `status: trialing` at creation | If Stripe applies different authentication or tax logic to trialing subscriptions than active ones, this mechanism could interact badly with the (also still-undecided) tax approach and the 3DS concerns Amara raised | **We assume** no meaningful behavioral difference beyond invoicing timing; not verified against Stripe's docs for this specific reuse pattern | Elena Sokolov |
| 3 | Whether writing off or one-time-collecting the 6 mid-cycle customers' currently-overdue invoice *outside* Stripe (design decision #7) is acceptable to Finance, versus something Finance expected to see reconciled through the new system | Ruth reconciles the first two months to the penny — an amount that's neither in the legacy engine's normal reporting nor in Stripe at all is exactly the kind of gap that could go unnoticed until an audit | **We assume** Finance will accept a one-off manual ledger entry for these 6 specific overdue amounts, separate from the Stripe migration entirely | Ruth Calloway / Priya Nair |
| 4 | Whether resetting the dunning retry count to zero for these 6 customers (rather than somehow preserving "retry 3 of 4") is an acceptable customer-experience/collections tradeoff, or whether leadership wants a manual, white-glove outreach for this specific small cohort instead of an automated reset | This directly affects real revenue recovery odds for exactly the customers already showing payment friction — the riskiest cohort to get wrong silently | **We assume** an automated reset (Stripe Smart Retries takes over fresh) is acceptable for all 6 — not ratified; given it's only 6 people, a manual outreach alternative is cheap and worth leadership's explicit choice | Amara Diallo / Priya Nair |
| 5 | Whether ANNUAL20 being retired as a redeemable code (decision #3) has any contractual/marketing implication (e.g., a support agent's ability to re-grant it to a saved-from-churn customer later) | If Marketing or Support ever wanted to re-offer ANNUAL20 as a save tactic, removing its redeemable form forecloses that without anyone deciding to | **We assume** ANNUAL20 is fully retired going forward, matching its "last year's push" framing | Chloe Baptiste |
| 6 | Whether combining `trial_end` (used purely as our renewal-date anchor) with `cancel_at_period_end: true` on the same subscription produces a clean cancellation with zero invoices at the anchor date, or whether Stripe requires at least one real billing cycle to elapse before `cancel_at_period_end` takes effect | If the combination doesn't behave as a clean pass-through, one of these 3 already-canceling customers could be invoiced once more before the cancellation honors — a real charge a customer explicitly opted out of | **We assume** the combination works as a clean pass-through with no invoice; not verified against Stripe's docs for this specific combination | Elena Sokolov |
| 7 | Whether Stripe Tax is actually enabled and registered for all relevant jurisdictions (US economic-nexus states, CA GST/HST provinces, EU VAT) on Streamhaven's account before the canary batch's first invoice | Registering/enabling Stripe Tax across jurisdictions has its own setup lead time separate from the API flag — assuming it's "just a config toggle" could quietly slip the tax decision past the first invoice despite Ruth's hard requirement | **We assume** Stripe Tax setup/registration is started immediately, in parallel with Wave 1, so it's live well before the canary — not yet confirmed as scheduled | Ruth Calloway / Elena Sokolov |
| 8 | Wren's requirement that business dimensions (acquisition channel, Founders/legacy tag, discount-window progress) survive the move is framed as a one-way door decided *before* Wave 2 creates 100 live subscriptions — acquisition_channel specifically is not attached as metadata anywhere in this draft or the Wave 1 draft | If it's warehouse-join-only and the join breaks or is delayed, CAC-payback dashboards die at cutover exactly as Wren warned, with no Stripe-side fallback | **We assume** `customers.acquisition_channel` is added to the Wave 1 customer-enrichment step's metadata (extending Wave 1 ambiguity #3) so it rides in Stripe metadata *in addition to* the warehouse join, rather than depending on the join alone | Wren Tanaka / Elena Sokolov |
| 9 | Whether Stripe Data Pipeline exposes an ongoing, decrementing "periods remaining" value for a `duration: repeating` coupon as it counts down invoice by invoice, or only ever the discount amount actually applied on each invoice | Wren explicitly wants month-by-month discount-window visibility for forecasting (START25's wind-down); `metadata.remaining_periods_at_migration` (design decision #2) is a fixed snapshot at migration, not a live counter | **We assume** the warehouse computes "periods remaining today" itself (snapshot value minus elapsed billing cycles since migration) rather than expecting Stripe's export to provide it directly — not yet confirmed against Stripe Data Pipeline's actual coupon fields | Wren Tanaka |
| 10 | Historical invoice backfill (Dev Kapoor/Wren, `summaries/03`) — whether pre-cutover invoice history gets recreated in Stripe or stays warehouse-only — is unresolved, but this draft's subscription-creation calls are the moment that decision would need to be executed, if it's executed at all | Without an explicit answer, revenue dashboards risk the "cliff at cutover" Dev Kapoor flagged, and nobody currently owns closing that gap between Wave 2's design and Pillar 4's reporting design | **We assume**, absent a decision, that Wave 2 does **not** backfill historical invoices into Stripe (only forward invoices from migration onward) and that Pillar 4 is responsible for stitching pre/post-cutover history at the warehouse layer — this is a working default, not a ratified answer | Wren Tanaka / Sam Okafor |

---

*Next draft: reporting requirements (Pillar 4), per `summaries/03`, building on the
`metadata.migration_billing_anchor` / product-catalog / coupon-variant decisions made here — plus the
business-dimension metadata, discount-window visibility, and invoice-backfill questions this draft
surfaced but could not close alone.*
