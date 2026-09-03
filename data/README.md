# Data Dictionary

All data in this folder is **fictional**, generated for this assessment. Card numbers are Stripe's
public test numbers (e.g., `4242…`). Any resemblance to real persons or accounts is coincidental.

**Conventions**: ISO 8601 UTC timestamps, money in **minor units** (cents), snake_case columns,
ISO-2 country codes, ISO-4217 currencies.

**Scenario date**: the migration project kicks off **2025-03-10**. Renewal dates are spread around
that date deliberately.

## Files

### `customers.csv` — internal customers table (100 rows)

| Column | Meaning |
|---|---|
| `customer_id` | Streamhaven ID (`CUST-####`) — the legacy identifier |
| `email`, `first_name`, `last_name` | Contact info |
| `country` | ISO-2 country |
| `billing_currency` | USD / CAD / EUR |
| `preferred_language` | en / fr / de |
| `acquisition_channel` | organic / paid_search / referral / partner |
| `is_business` | true for business accounts |
| `tax_id` | VAT-style tax ID (business accounts only) |
| `tax_exempt` | true where finance flags the account exempt |
| `account_status` | active / churned (churned = subscription already canceled) |
| `created_at` | Signup date. Founders cohort signed up 2005–2012 |

### `subscriptions.csv` — internal subscriptions table (100 rows, one per customer)

| Column | Meaning |
|---|---|
| `subscription_id` | Streamhaven ID (`SUB-####`) — the legacy identifier |
| `customer_id` | → `customers.customer_id` |
| `plan` | `basic` / `pro` / `founders` |
| `billing_interval` | `monthly` / `annual` (Founders is monthly-only; annual = 10× monthly, 2 months free) |
| `currency` | Matches the customer's billing currency |
| `price_amount_cents` | Per-period price in minor units, **excluding coupon effects** |
| `status` | `active` / `trialing` / `past_due` / `canceled` |
| `created_at` | Subscription start |
| `current_period_start` / `current_period_end` | Current billing period |
| `cancel_at_period_end` | true = cancellation queued for period end |
| `canceled_at` | Set when fully canceled |
| `auto_renew` | Self-explanatory |
| `coupon_code` | Active coupon, if any (see coupon definitions in `corpus/`) |
| `discounted_periods_remaining` | For repeating coupons: periods of discount left. Empty for `forever`/one-shot coupons |
| `pending_plan_change` | Scheduled downgrade, e.g. `pro_to_basic`, effective at period end |
| `pending_plan_effective_at` | When the queued change lands |
| `dunning_stage` | `retry_1..retry_3` / `final_notice` — set when payment is failing |
| `failed_payment_attempts` | Count of failed charges this cycle |
| `days_past_due` | Days into the 30-day dunning cycle |
| `next_retry_at` | When the in-house engine will retry |
| `trial_end` | For trialing subscriptions |

### `payment_methods.csv` — internal payment methods table (100 rows, one per customer)

| Column | Meaning |
|---|---|
| `payment_method_id` | Streamhaven ID (`PM-####`) — the legacy identifier |
| `customer_id` | → `customers.customer_id` |
| `atlaspay_token` | AtlasPay token used for all billing (`atp_tok_…`) |
| `card_brand` / `card_last4` | Display metadata |
| `exp_month` / `exp_year` | Card expiry (check these against the scenario date) |
| `cardholder_name` / `billing_postal_code` | Used for AVS/verification |
| `status` | `active` / `expired` |
| `created_at` | When the card was tokenized |

### `atlaspay_card_vault_export.csv` — prepared for the Wave 1 upload

The mapping AtlasPay prepared from its vault: one row per card, keyed by AtlasPay token, including
the **raw PAN** (fictional test numbers). This is the file that must travel to Stripe over a secure
channel. Columns: `atlaspay_token`, `customer_id`, `payment_method_id`, `pan`, `exp_month`,
`exp_year`.

### `stripe_migration_response_sample.csv` — what Stripe sends back

Sample from a **15-token test batch** processed by Stripe on 2025-03-06 (this is the shape of the
real Wave 1 response, which will cover all rows). The file is purely an **old → new ID mapping** —
nothing else comes back:

| Column | Meaning |
|---|---|
| `old_customer_id` | The `customer_id` we submitted |
| `new_customer_id` | New Stripe customer (`cus_…`) created by the import |
| `old_payment_method_id` | The `payment_method_id` we submitted |
| `new_payment_method_id` | New Stripe payment method (`pm_…`) |

**Failures are silent.** The file contains one row per card that *migrated*. Cards that fail do
not appear at all — there is no status or error column. Identify them by diffing the submitted
references against the returned rows. Of the 15 tokens in our test batch, **12 came back**; the 3
that didn't were two expired cards and one account the issuing bank had closed (we know that only
because we cross-checked AtlasPay's card records).

## Price book (effective 2021 re-tiering)

| Plan | Monthly | Annual |
|---|---|---|
| Basic — USD | 499 | 4990 |
| Basic — CAD | 699 | 6990 |
| Basic — EUR | 499 | 4990 |
| Pro — USD | 2999 | 29990 |
| Pro — CAD | 3999 | 39990 |
| Pro — EUR | 2999 | 29990 |
| Founders (USD, monthly only) | 499 | — |

Annual is priced at ten monthly payments ("two months free"). Founders keeps its 2005 price-locked
$4.99 and predates both annual billing and multi-currency.
