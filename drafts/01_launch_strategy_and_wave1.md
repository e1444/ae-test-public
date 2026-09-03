# Draft: Pillar 1 (Launch Strategy) + Pillar 2 (Wave 1 — Customer & Payment-Method Migration)

**Status:** Working draft for `PRD.md`. Scope is intentionally limited to pillars 1 & 2 — Wave 2
billing mechanics (dunning, downgrades, coupons, trials) and reporting are covered separately and
only referenced here where they constrain sequencing.

**Inputs used:** `summaries/00`–`06` (all corpus docs), `data/README.md`, and verified row counts
from `data/*.csv` (100 customers/subscriptions/payment methods; 8 Founders; 6 mid-dunning; 5 queued
pro→basic; 4 trialing; test batch 15→12 with 3 silent failures).

**Key constraints this draft is built against:**
- Kickoff **March 10, 2025**; Wave 2 must be complete and *soaked* before the **June 1–15** renewal
  wave (Dana, Sam).
- **Zero double-billing**, **renewal dates preserved**, as mechanisms not intentions (Priya).
- Rollout pacing is explicitly a **support decision** — phased, with checkpoints, not a big bang
  (Amara/Marcus).
- Wave 1 must not change anything customer-facing — no charge is ever attempted through Stripe
  during Wave 1 (Jonas/Sam).

---

## Pillar 1 — Launch Strategy

### Guiding principles
1. **Decouple "card data exists in Stripe" from "customer bills in Stripe."** Wave 1 is invisible
   to customers and can run for the whole population up front. Wave 2 is what actually flips billing
   and is what gets batched.
2. **A subscription is either billing in the legacy engine or in Stripe — never both, never neither**,
   at any point in time. This is enforced by an explicit state, not a convention.
3. **Every batch is a checkpoint, not a commitment.** Nothing proceeds to the next batch until the
   previous one is reconciled and support-clean.
4. **Soak time is a deliverable, not a buffer.** The plan must show weeks of stable, unattended
   operation before the June 1–15 window, not a cutover that finishes days before it.

### "Live on Stripe" — the state machine
Today neither `customers` nor `subscriptions` has any field answering "who bills this subscription
tonight." That's a direct gap flagged by Sam Okafor and Elena Sokolov. Proposed states, held on
`subscriptions.billing_system`:

| State | Meaning | Who may charge |
|---|---|---|
| `legacy` | Default / pre-migration | Legacy engine only |
| `wave1_complete` | Stripe customer + payment method exist; no Stripe subscription yet | Legacy engine only (Wave 1 has no billing effect) |
| `migrating` | Stripe subscription/price/coupon objects created for this customer; not yet verified | **Neither** — legacy engine's charge job explicitly excludes this state; Stripe subscription is created with billing paused (see below) until verification passes |
| `stripe_live` | Verified: Stripe subscription's next invoice matches expected amount, date, and discount state | Stripe only |
| `migration_failed` | Verification failed or manual hold | Legacy engine only (fall back), flagged for manual review |

**How the Stripe-side pause actually works (closing the "see below" reference above):** the
Stripe subscription is created with `pause_collection: {"behavior": "void"}` set at creation time,
so Stripe's own invoicing engine is structurally incapable of generating or paying an invoice for it
regardless of what `current_period_end` computes to — this is what makes "Neither system can charge"
a Stripe-side mechanism for `migrating`, not just a legacy-side query filter. The flip from
`migrating` to `stripe_live` clears `pause_collection` in the *same* transaction that flips
`subscriptions.billing_system`, so there is never a window where both systems are simultaneously
eligible to bill, or where neither is — this is the concrete answer to Ruth Calloway's ask for one
unambiguous source of truth during the cutover instant itself.

A subscription only reaches `stripe_live` after an automated **pre-flight check** compares the
upcoming Stripe invoice preview (amount, currency, `next_payment_attempt`) against the expected
values computed from the legacy row. This is the "mechanism, not intention" Priya asked for. Note
this check only confirms *configuration* before the fact — it does not observe a real charge
succeeding. Closing that loop needs a minimal Stripe webhook consumer (`invoice.payment_succeeded`,
`invoice.payment_failed`, `invoice.payment_action_required`) wired in for Wave 2 cutover itself,
ahead of the fuller Stripe→Streamhaven sync design Sam asked for more broadly (open question,
Wave 2 doc).

**Mutual exclusion (the actual double-charge guard):** the legacy billing cron's query gets a hard
`WHERE billing_system = 'legacy' OR billing_system = 'wave1_complete'` filter — it physically cannot
select a `migrating`/`stripe_live` row. Stripe never sees a legacy subscription at all, so the only
side that can double-charge is the legacy engine continuing to run against an already-migrated
customer; that's the side we gate. Additionally, every Stripe object we create carries our internal
ID in `metadata` and is created with a caller-supplied idempotency key derived from that ID, so a
retried migration job can never create a duplicate Stripe customer/subscription.

### Batching & sequencing
Rollout is phased by **risk profile**, not customer count, per Marcus's ask for pacing and
checkpoints:

| Batch | Population | Rationale |
|---|---|---|
| **Canary** (~5 customers) | Active, monthly, no coupon, no pending change, no trial, one per currency | Proves the end-to-end mechanism (Wave 1 already done for these) with minimum blast radius before trusting it with anyone complex |
| **Batch 1** (~25) | Active, monthly, no coupon/pending-change complications | Largest simple cohort; validates scale before adding edge cases |
| **Batch 2** (~25) | Active, annual, and customers with a simple one-shot/forever coupon (`TAKE5`, `ANNUAL20`) | Adds renewal-date-preservation risk (annual) and static-discount risk under real volume |
| **Batch 3** (~25) | Customers with repeating/partial-window coupons (`START25`, `FRIEND50`, `STREAM30`) and trialing customers | Adds in-flight-discount carry-over and trial-boundary risk; requires Wave 2 coupon design (separate doc) to be signed off first |
| **Batch 4** (remainder) | Customers with pending plan changes (5) and currently past-due/dunning (6) | Highest-risk cohort — requires the dunning-treatment and downgrade-representation decisions (open, see Wave 2 doc) to be finalized before this batch is scheduled |
| **Founders track** (8) | Run as its own small batch, scheduled after the canary proves the mechanism, with the dedicated comms track from `summaries/04`/`06` | Not deferred to "last" — trust matters more than risk-avoidance for this cohort; low complexity (monthly, no coupons) but high sensitivity |

**Verified against `data/subscriptions.csv`** (the "~25"s above were estimates; exact,
non-overlapping segment sizes are):

| Segment | Count | Interval |
|---|---|---|
| Founders | 8 | monthly |
| Dunning (`past_due`) | 6 | 5 monthly / 1 annual |
| Pending pro→basic downgrade | 5 | monthly |
| Trialing | 4 | monthly |
| Repeating/partial coupons (`START25` 6, `FRIEND50` 4, `STREAM30` 5) | 15 | monthly |
| Static coupons (`ANNUAL20` 3, `TAKE5` 2) | 5 | 3 annual / 2 monthly |
| Plain — annual, no coupon/pending/trial | 19 | annual |
| Plain — monthly, no coupon/pending/trial | 33 | monthly |
| Canceled/churned (excluded — see Pillar 2 ambiguity #5) | 5 | 2 annual / 3 monthly |
| **Total** | **100** | |

These segments don't overlap (no dunning/pending-change customer also carries a coupon or is
trialing), which is what makes the batch table above a clean partition. With canceled/churned
customers excluded, the batches should total **95**, not ~100: Canary (5, drawn from the
plain-monthly pool) + Batch 1 (remaining ~28 plain-monthly) + Batch 2 (19 plain-annual + 5
static-coupon = 24) + Batch 3 (15 repeating-coupon + 4 trialing = 19) + Batch 4 (11) + Founders
(8) = 95.

Each batch has an explicit **exit checklist** before the next one starts:
- 100% of the batch's subscriptions reached `stripe_live` or `migration_failed` (no stragglers stuck
  in `migrating`).
- Reconciliation report ties Stripe's recorded amounts to the legacy engine's expected amounts for
  every row in the batch.
- Blended and per-region auth rate for the batch's first charges is within an agreed tolerance of
  baseline (91.4% blended / 93% NA / 88% EU) — a same-day explanation is required for any miss
  (per Priya's ask), regardless of whether it blocks progression.
- No unresolved P0/P1 support tickets traced to the batch.

### Indicative timeline (working backward from the June 1–15 freeze)
| Window | Activity |
|---|---|
| Mar 10–14 | Wave 1 for all ~100 customers (see Pillar 2) — no customer impact |
| Mar 17–19 | Canary batch (Wave 2) |
| Mar 20–26 | Batch 1 |
| Mar 27–Apr 2 | Batch 2 |
| Apr 3–9 | Batch 3 (contingent on coupon-carryover design being signed off) |
| Apr 10–18 | Batch 4 + Founders track (contingent on dunning/downgrade decisions) |
| Apr 19–May 31 | **Soak.** No further migrations. Monitor auth rate, reconciliation, support volume across at least one full monthly cycle for every migrated customer and confirm no annual renewal has occurred incorrectly |
| Jun 1–15 | Renewal freeze — no migration activity; only monitoring and support |

This leaves ~6 weeks of soak before the renewal wave, which is the "soak time, not a date on a
slide" leadership asked for.

⚠️ This schedule rests on two unconfirmed assumptions — see items 1 and 5 in the ambiguities table
below: that the tax decision lands before Mar 17, and that pre-flight invoice-preview checks (rather
than an observed real renewal) are acceptable proof of correctness for annual subscribers who won't
renew again before June.

### Rollback (scoped to launch mechanics; full risk register lives in the risk section of `PRD.md`)
Rollback granularity is **per-subscription**, not per-batch, because batches contain independent
customers:
- **Before `stripe_live`:** trivial — set `billing_system` back to `legacy`/`wave1_complete`; the
  Stripe subscription (if created) is canceled without proration; nothing was ever charged there.
- **After `stripe_live` but within the same billing period:** the harder case. Rollback means
  canceling the Stripe subscription and resuming the legacy engine for that customer — only safe if
  no Stripe invoice has been paid yet for the period in question. If one has, "rollback" becomes a
  reconciliation/credit problem, not a technical one, and must be handled by Finance, not silently
  reversed by a script.
- **General rule:** the migration tooling never deletes legacy billing capability for a customer
  until `stripe_live` is confirmed **and** has survived one full renewal — legacy stays dormant-but-
  present as a fallback, not decommissioned per-customer, until that point.

### Ambiguities requiring leadership clarification (Pillar 1)

Distinct from the engineering-owned "Open decisions" below — these need a **business/leadership**
answer before the plan can be trusted, not just an engineering design choice. Where we'd otherwise
proceed silently, the assumption we're actually building against is written out explicitly.

| # | Ambiguity | Why it matters | Assumption if we proceed without an answer | Needs sign-off from |
|---|---|---|---|---|
| 1 | Tax approach is still undecided (Ruth, Mar 7), but the canary batch is scheduled to create Stripe's **first invoices** as early as Mar 17 | Leadership required tax to be settled *before the first Stripe invoice*, not retrofitted — as drafted, the timeline conflicts with that constraint | **We assume** the canary is restricted to a domestic (US, non-VAT, non-exempt) customer so no tax logic is exercised, and that no batch touching CA/EU or business/exempt accounts starts until tax approach is ratified — this ordering has not been confirmed as acceptable, and no date has been given for the tax decision | Priya Nair / Ruth Calloway |
| 2 | No one has named who approves progression from one batch to the next | Amara asked for "a checkpoint after each batch" — a checkpoint without a named approver is a report nobody is accountable for acting on | **We assume** Sam Okafor (engineering) and Amara Diallo (support) jointly sign off, with Priya's team independently signing off the reconciliation figure | Dana Whitfield (to name the approver(s)) |
| 3 | No numeric auth-rate tolerance was ever set — Priya only asked for "a same-day explanation" if the number moves | Without a threshold, "moves" is undefined and any batch's result can be argued either way after the fact | **We assume** a provisional tolerance of ±2 points blended / ±3 points per region before a batch is paused pending explanation — not validated with Priya | Priya Nair |
| 4 | No incident-severity definition exists for the "no unresolved P0/P1 tickets" exit criterion | Support may have an existing taxonomy we're not using, or none at all, making the exit criterion unenforceable as written | **We assume** Support's existing (undocumented-to-us) incident severity scale applies unchanged | Amara Diallo |
| 5 | The soak plan's claim to "confirm no annual renewal has occurred incorrectly" is unobservable for any annual subscriber whose `current_period_end` falls after the June freeze — most of them, by definition, won't renew again before June | Our only evidence of correctness for those subscribers is the pre-flight invoice-preview match, never an actual charge, before the real June renewal hits them | **We assume** the pre-flight invoice-preview check is accepted as sufficient proof for annual subscribers who can't be soak-validated by a real renewal in this window — not confirmed as an acceptable risk position | Priya Nair / Sam Okafor |
| 6 | Batch 4's population depends entirely on the (currently unresolved) dunning-treatment and downgrade-representation decisions — it's possible mid-dunning customers shouldn't be migrated as a scheduled batch at all | The batch plan may not be the right shape for this cohort once that decision lands | **We assume**, pending that decision, mid-dunning customers are migrated individually/manually rather than as a scheduled batch — not yet validated | Sam Okafor (per the Wave 2 dunning-decision owner) |
| 7 | Rollback "after a charge has already succeeded" is called out as a Finance problem, but Finance has never defined what that process actually is (credit note? refund? manual ledger entry?) | The rollback path most likely to be needed — something goes wrong right after go-live — currently has no runbook | **We assume** Ruth Calloway's team would handle this ad hoc, via a manual credit note per incident, until a real process exists | Ruth Calloway |
| 8 | The statement descriptor (`ATLASPAY*STREAMHAVEN` today) has no decided replacement, but the canary batch is scheduled to produce Stripe's first real charge as early as Mar 17–19, and Amara/Elena both require comms to precede the *first* charge under any new descriptor | If the descriptor isn't decided and pre-communicated before the canary, we breach that commitment on the very first Stripe charge and risk the "was I hacked?" ticket spike Amara described (baseline ~40/month on routine descriptor glitches alone) | **We assume** engineering brings descriptor options (character-limit/prefix constraints) to Chloe in time to decide and pre-announce before Mar 17 — not yet scheduled anywhere | Chloe Baptiste (final pick) / Elena Sokolov (options) |
| 9 | EU auth rate (88% baseline) is explicitly tied by Priya to 3DS friction, and Marcus/Chloe committed to a pre-cutover "your bank may ask you to approve your next payment" email for the EU base — but no batch above is gated on that email having gone out first | A batch containing EU customers could hit its first Stripe renewal without the agreed pre-warm, understating how much of any auth-rate dip is migration-related vs. normal 3DS friction, and breaking Support's ask for comms-before-surprise | **We assume** the EU pre-warm email is sent before Batch 2 (the first batch containing EU customers per the segment table below) and is a formal gate, not best-effort — not yet scheduled | Chloe Baptiste / Amara Diallo |
| 10 | Dana approved temporary support coverage and improved macros "contingent on the plan showing timing" — this plan never states when that coverage starts or ends | Temp coverage arriving after the ticket spike it's meant for defeats the point; Support needs staffing lead time, not a same-week scramble | **We assume** temp coverage ramps up to cover the canary + Batch 1 window (the first real Stripe charges) and stays through Batch 4, tapering once soak begins — not yet confirmed with Amara | Amara Diallo / Dana Whitfield |

### Open decisions (Pillar 1)
| # | Decision | Options | Recommendation | Owner |
|---|---|---|---|---|
| 1 | Batch size/cadence | Big bang vs. weekly batches of ~25 vs. daily micro-batches | Weekly batches as above — matches Marcus's "checkpoint after each batch" ask without dragging past soak time | Amara Diallo + Sam Okafor (joint, per Amara's framing that pacing is a support call) |
| 2 | Founders sequencing | First (build trust early) vs. last (de-risk) vs. dedicated mid-project batch | Dedicated batch right after canary (above) | Dana Whitfield (per leadership checkpoint — Founders treatment needs a named decision-maker) |
| 3 | Pre-flight check strictness (what blocks `stripe_live`) | Amount+date match only vs. also requiring a successful $0 auth/SetupIntent confirmation on the card | Add a SetupIntent-based card verification for annual/high-value subs only, to catch dead cards before the real first charge | Sam Okafor / Elena Sokolov |
| 4 | Statement descriptor string | Clean `STREAMHAVEN` (Chloe's stated preference) vs. a processor-prefixed form if Stripe/acquirer conventions require one vs. retaining a reference to the old descriptor for a transition period | Engineering to confirm Stripe's actual character-limit/prefix constraints first; if a clean `STREAMHAVEN` is technically available, recommend it — it's what Marketing asked for and removes a processor name customers never chose | Chloe Baptiste (ratifies; Elena Sokolov supplies constraints) |

---

## Pillar 2 — Wave 1: Customer & Payment-Method Migration

### End-to-end flow (verified mechanics from `summaries/01`, `02`)
1. AtlasPay produces the vault export (`atlaspay_card_vault_export.csv`): token, `customer_id`,
   `payment_method_id`, PAN, expiry — already staged in `data/`.
2. Streamhaven PGP-encrypts the file with Stripe's issued public key. Plaintext is generated only in
   an isolated job, never written to shared storage, and deleted immediately after encryption.
3. The encrypted file + a manifest (row count, column shape, batch ID) is pushed over SFTP to
   Stripe's migration inbox.
4. Stripe creates one Customer + one PaymentMethod per successfully processed row.
5. Stripe drops a response file back on the SFTP: `old_customer_id, new_customer_id,
   old_payment_method_id, new_payment_method_id` — **one row per success only, no status/error
   column.**
6. Streamhaven diffs submitted references against returned references. Anything submitted and not
   returned is a **silent failure** — cross-check against `payment_methods.status` /
   AtlasPay's own records to learn *why* (expired, bank-closed, etc.), same as the Mar 6 test batch
   (15 in, 12 back: 2 expired, 1 bank-closed).

### Timing / polling
Stripe's actual processing SLA for this batch import is **not documented in anything we have** — it
is an unverified assumption whether the response arrives synchronously, same-day, or after some
processing window. **Action item:** confirm the real turnaround from Stripe before relying on any
specific wait time. Until confirmed, the migration job polls the SFTP outbox on a fixed interval and
only declares unmatched rows "failed" once a conservative window (default: 48h) has elapsed *and*
Stripe has confirmed the batch is closed — never based on absence alone at T+0.

### The "row conflates customer + payment method" problem
Because the response file returns a single row per success covering **both** the new customer and
the new payment method, we cannot assume from Stripe's side which part of a failed row succeeded
partially (e.g., customer created but card rejected). Rather than guess:
- For every row that comes back **failed** (absent from the response), Streamhaven explicitly
  creates the Stripe Customer object directly via the API (idempotent on our `customer_id`,
  metadata-tagged).

  ⚠️ **Unverified assumption:** our idempotency key only protects against *our own* retried calls —
  it does **not** protect against Stripe's bulk-import path having already created a Customer for
  this row, under a different and unknown ID, before the row failed. We are proceeding on the
  assumption that a failed row has zero footprint on Stripe's side (the FAQ's plain reading of "the
  3 absent rows are the failures"), but this has not been confirmed by Stripe. If it's wrong, this
  remediation step creates an undetectable duplicate customer record — see ambiguity #2 below.
- The payment method is then collected fresh from the customer via a hosted **SetupIntent** flow
  (see below), never assumed to exist.

### Failed-card remediation journey
1. Customer's `payment_methods.migration_status` = `failed`; `customers.billing_system` stays
   `legacy` (never advances past Wave 1 for this person until remediated).
2. Support-approved email triggers within 24h of a batch closing: "your card on file couldn't be
   moved — update it here" (per `summaries/06`), linking to a mobile-friendly, subscription-scoped
   hosted update page.
3. That page creates a Stripe SetupIntent against the (already-created, per above) Stripe Customer,
   collects a new card, and on success writes the new `pm_...` back into our mapping table,
   flipping `migration_status` to `migrated`.
4. Until remediated, the customer's subscription simply never advances into Wave 2 batching — they
   stay on the legacy engine, which still has their AtlasPay token and keeps billing normally. No
   customer is ever left with *no* working payment method as a side effect of migration.

### Data model changes
New/changed columns and a new mapping table (kept separate from the legacy tables rather than
overloading them, for clean auditability and idempotency):

```sql
-- new mapping table, one row per successfully migrated card
CREATE TABLE stripe_payment_method_migrations (
  payment_method_id      TEXT PRIMARY KEY REFERENCES payment_methods(payment_method_id),
  customer_id            TEXT NOT NULL REFERENCES customers(customer_id),
  stripe_customer_id     TEXT NOT NULL,
  stripe_payment_method_id TEXT NOT NULL,
  migration_status       TEXT NOT NULL CHECK (migration_status IN ('migrated','failed','remediated')),
  source                 TEXT NOT NULL CHECK (source IN ('bulk_import','manual_remediation')),
  batch_id               TEXT NOT NULL,
  migrated_at            TIMESTAMPTZ,
  created_at             TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE customers ADD COLUMN stripe_customer_id TEXT;
ALTER TABLE customers ADD COLUMN billing_system TEXT NOT NULL DEFAULT 'legacy';
ALTER TABLE subscriptions ADD COLUMN billing_system TEXT NOT NULL DEFAULT 'legacy';
ALTER TABLE subscriptions ADD COLUMN stripe_subscription_id TEXT;

-- audit ledger of each Wave 1 batch submitted to Stripe
CREATE TABLE wave1_migration_batches (
  batch_id        TEXT PRIMARY KEY,
  submitted_at    TIMESTAMPTZ NOT NULL,
  row_count       INT NOT NULL,
  response_received_at TIMESTAMPTZ,
  matched_count   INT,
  failed_count    INT,
  status          TEXT NOT NULL CHECK (status IN ('submitted','partial','closed'))
);
```

`billing_system` is duplicated on both `customers` and `subscriptions` deliberately: the customer-
level flag reflects Wave 1 (card presence), the subscription-level flag reflects Wave 2 (who bills
this subscription) — a customer can be `wave1_complete` while individual subscriptions are still
`legacy`.

### Security
- File never leaves the encrypted-SFTP path — no email, ticket, or laptop copy at any stage (already
  a decided constraint, restated here as an implementation requirement).
- PGP key managed via [secrets manager], rotated per Stripe's guidance; access to the vault-export
  generation job restricted to the migration service account.
- All access to `atlaspay_card_vault_export.csv`-equivalent data in any environment is logged.
- Response file (contains only Stripe IDs, no PAN) can be handled with standard access controls —
  it is not itself PCI-scoped data.

### Sample Stripe API calls

**Remediation-path customer creation** (used only for rows absent from the bulk-import response,
per the "don't assume partial state" decision above):

```json
POST /v1/customers
{
  "email": "clara.dubois10@example.com",
  "name": "Clara Dubois",
  "metadata": {
    "streamhaven_customer_id": "CUST-0010",
    "migration_source": "manual_remediation",
    "migration_batch_id": "wave1-2025-03-10"
  }
}
```
*Idempotency-Key header: `wave1-cust-CUST-0010` — guarantees a retried remediation job never
creates a duplicate customer.*

**SetupIntent for the failed-card remediation flow:**

```json
POST /v1/setup_intents
{
  "customer": "cus_<created_above>",
  "payment_method_types": ["card"],
  "usage": "off_session",
  "metadata": {
    "streamhaven_customer_id": "CUST-0010",
    "streamhaven_payment_method_id": "PM-0010"
  }
}
```
*`usage: off_session` is the non-obvious field — it tells Stripe this payment method will be
charged later without the customer present (i.e. on renewal), which affects the authentication
Stripe requests at setup time (may trigger 3DS now instead of at the first renewal charge, which is
the safer place for it to happen).*

### Ambiguities requiring leadership/Stripe clarification (Pillar 2)

| # | Ambiguity | Why it matters | Assumption if we proceed without an answer | Needs sign-off from |
|---|---|---|---|---|
| 1 | Stripe's real turnaround SLA for the bulk-import response file is undocumented in anything we have — the FAQ itself flags this as unverified | Declaring a row "failed" too early risks a false failure and unnecessary remediation contact; too late delays all of Wave 1 | **We assume** a 48h wait plus an explicit Stripe batch-closed signal before declaring any row failed (see "Timing / polling" above) | Elena Sokolov, to confirm with Stripe's migration team |
| 2 | Whether Stripe's bulk import can partially create a Customer object for a row whose PaymentMethod ultimately fails — i.e. does "absent from the response" always mean *nothing* was created? | If a Customer was silently created, our remediation flow's fresh `POST /v1/customers` call could produce a **second, permanent duplicate Stripe customer** for the same person, undetectable without a dedicated search | **We assume** failed rows have zero footprint on Stripe's side — not confirmed by Stripe and should be verified before remediation logic ships | Elena Sokolov / Stripe support |
| 3 | The AtlasPay vault export (`atlaspay_card_vault_export.csv`) carries only token, customer ref, PAN, and expiry — no email, name, tax ID, or address | Stripe Customer objects created via the bulk import may come back as bare shells with no contact/tax info; email/name are needed for receipts, tax ID for the still-undecided reverse-charge VAT handling before the first invoice | **We assume** a follow-up enrichment call is required per migrated customer (email/name/tax ID/address from our own `customers` table, keyed by `new_customer_id`) — this step isn't designed anywhere yet and must land before Wave 2 pricing/tax logic runs | Jonas Meyer / Elena Sokolov |
| 4 | Whether the manifest/file-shape Elena prototyped "against the docs" has actually been confirmed by Stripe, versus just checked against public documentation | A rejected or misparsed batch on the real ~100-row submission would stall all of Wave 1 | **We assume** the prototyped shape is correct; recommend a small confirmation submission with Stripe support before the real batch | Elena Sokolov |
| 5 | Whether already-churned/canceled customers (5 in the current data) are in scope for migration at all | They have no future billing — migrating their cards is added PCI/security surface for no operational benefit | **We assume** churned/canceled customers are **excluded** from both Wave 1 and Wave 2 entirely — nobody has stated this explicitly | Sam Okafor / Dana Whitfield |
| 6 | No threshold has been set for pausing Wave 1 outright if the real failure rate is much higher than the test batch's 20% (3/15) | The test rate may not be representative; treating every failure identically regardless of volume risks missing a systemic problem (e.g. a bad encryption step) rather than 100 individual card problems | **We assume** any failure rate above 10% of a submitted batch triggers a hold-and-investigate before continuing per-card remediation | Elena Sokolov |
| 7 | This draft only designs self-serve *card update* for the failed-migration remediation case (Stripe SetupIntent, above). Amara's broader ask — ordinary add/replace-card and cancel, for customers who never had a failed migration, once Stripe is system of record — has no owner or design here | Today's self-serve flows hit the legacy engine directly; if they aren't repointed at Stripe before a customer reaches `stripe_live`, either the buttons silently do nothing, or Support's macro library goes stale the moment the first batch goes live | **We assume** the remediation SetupIntent flow is generalized into the permanent self-serve "update card" flow (same mechanism, not a one-off), and cancellation is handled by canceling the Stripe subscription directly once `stripe_live` — but this hasn't been scoped as its own workstream | Elena Sokolov / Amara Diallo (macro library must match) |
| 8 | The SetupIntent's `usage: off_session` addresses 3DS at *remediation* time, but says nothing about 3DS at ordinary *renewal* time for the ~20% of the base that's EU — Priya named this as an explicit auth-rate risk, separate from the remediation flow | If renewal-time 3DS challenges aren't configured deliberately (e.g. off-session recurring-payment exemptions vs. forced re-auth every cycle), an EU renewal during an unattended overnight window can fail silently and become the exact kind of ticket Amara forecasted | **We assume** Stripe's standard off-session recurring-payment exemption flow is used for renewals (issuer-side 3DS only on issuer challenge, not forced every cycle), paired with the EU pre-cutover pre-warm email — not yet verified against Stripe's SCA/3DS configuration options for our account | Elena Sokolov |

### Open decisions (Pillar 2)
| # | Decision | Options | Recommendation | Owner |
|---|---|---|---|---|
| 1 | Wait window before declaring a row "failed" | Fixed 48h vs. wait for Stripe's explicit batch-closed signal | Wait for explicit signal if Stripe's import provides one; confirm via Stripe support/docs before build | Elena Sokolov (verify against Stripe docs, per FAQ's own "not yet verified" flag) |
| 2 | Whether to re-attempt failed rows in a later Wave 1 batch (e.g., customer gets a new card before remediation) | Auto-resubmit vs. always route through the manual SetupIntent remediation flow | Manual remediation flow only — avoids re-triggering the same silent-failure ambiguity from a second bulk import | Jonas Meyer |
| 3 | SCA/3DS posture on remediation SetupIntents | `usage: off_session` (may prompt 3DS immediately) vs. deferring auth entirely to first renewal | `off_session` as above — surfaces a dead/declined card during a support-assisted flow rather than at an unattended renewal | Elena Sokolov / Amara Diallo |
| 4 | Ongoing self-serve card/cancel flow implementation | Rebuild custom UI against Stripe's APIs (Payment Element / SetupIntents) vs. adopt Stripe-hosted equivalents (Customer Portal) | Recommend Stripe Customer Portal for cancel + card update — matches Amara's "options exist at every depth, it's scoping not invention" framing and ships faster than a custom rebuild, at the cost of less control over the exact UI Marcus flagged (mobile-friendly, subscription-specific) | Amara Diallo (owns whether the portal's UX meets Marcus's bar) |

---

*Next drafts: Wave 2 billing & subscription migration (plans, prices, coupons, dunning, downgrades,
trials) and reporting requirements, per `summaries/01`–`04`.*
