# Engineering Wiki Excerpt: Relay FAQ (living document)

**Page:** `wiki/eng/relay-faq` · **Owner:** Jonas Meyer · **Last updated:** 2025-03-07

Answers to the questions that keep landing in #proj-relay. Facts here are verified against Stripe's
documentation and our AtlasPay integration notes unless marked otherwise. This page records
*decided* things. Open decisions live in the PRD.

---

## Q: What exactly is the Stripe card import process? Step by step.

A: Verified against docs and our test batch:

1. AtlasPay produces the vault export (token, customer ref, PAN, expiry) — done, it's in the data
   folder.
2. We encrypt the file with PGP using the public key Stripe issued for our account.
3. We upload the encrypted file plus a manifest over **SFTP** to Stripe's migration inbox. The
   manifest declares the row count and column shape.
4. Stripe processes the batch and creates a Customer and a PaymentMethod per row (cards only —
   this program is for **card data**; bank-based and wallet payment methods are not part of Wave 1
   and are out of scope entirely — we have none anyway).
5. Stripe drops a **response file** back on the SFTP. One row per **successfully migrated** card:

   ```text
   old_customer_id, new_customer_id, old_payment_method_id, new_payment_method_id
   ```

6. That's the whole file — an old→new mapping, nothing else. There is **no status or error column**:
   cards that fail to migrate simply do not appear in the response. Find them by diffing the
   references you submitted against the rows that returned. In our March 6 test batch: **15 tokens
   in, 12 back** — the 3 absent were two expired cards and one account closed by the issuing bank.
   Stripe won't tell you why; that comes from cross-checking AtlasPay's card records.
7. Sample from our March 6 test batch: `data/stripe_migration_response_sample.csv`
   (15 submitted, 12 returned — the 3 absent rows are the failures).

## Q: Are the PANs in the export real?

A: They're Stripe's published test numbers — the repo's data is fully fictional. Treat them as if
they were live PANs anyway: in production this file moves by encrypted SFTP only, never email,
never a ticket, never a laptop.

## Q: Why aren't we using Stripe's official migration service?

A: Decided at exec level (see Dana's Feb 24 email). We want repeatability, scheduling control, and
in-house tooling. Not up for debate.

## Q: Can we import a customer who is mid-dunning?

A: We don't think so — our working reading is Stripe has no way to represent "on retry 3 of 4 of a
30-day dunning cycle." **Not yet verified against their docs; someone needs to confirm before we
plan around it.** What we do about the six mid-cycle customers either way is a separate, open
decision — with one constraint for planning: the in-house dunning engine is being retired, not
migrated. Whatever the end state is, it runs on Stripe's side.

## Q: How do "scheduled downgrades" survive the migration?

A: Unknown — it's a design task. Facts: our `subscriptions` table queues plan changes
(`pending_plan_change` = `pro_to_basic`, `pending_plan_effective_at` = period end). Five customers
are queued, all effective in April 2025. Nobody has worked out what this looks like on Stripe's
side. It has to look like *something* before cutover: five people chose Basic and expect to be
billed as Basic.

## Q: How do in-flight coupon discounts survive?

A: Unsolved — and marketing is watching this one closely. Facts: five coupons live (see the
marketing review), and our `subscriptions` table carries `discounted_periods_remaining` for running
coupons, so we know exactly how much discount each subscriber has left. What we owe them is
unambiguous. How it carries over is the design question.

## Q: What does "live on Stripe" mean operationally?

A: **Not yet defined — this is a PRD deliverable.** What we know: ops needs per-subscription
visibility of where a customer bills, and Priya's line in the sand is zero double-billing. What
"live" means mechanically — and what guarantees it — is the design question.

## Q: Do we need webhooks?

A: Unresolved. The requirement is settled: after cutover, renewals, failures, and cancellations
happen on Stripe's side and have to land back in our systems — ops can't be blind, and Wren's
warehouse needs its feed (see her thread). The mechanism is a PRD question.

## Q: What about taxes?

A: **Open.** Finance hasn't settled the approach. Whatever the PRD assumes has to handle our EU
business accounts with VAT IDs and the few flagged tax-exempt, and must be working before the
*first* Stripe invoice — not retrofitted.

## Q: What's the deadline pressure?

A: The June renewal wave — a large cluster of annuals renews June 1–15. Wave 2 must be complete and
soaked before that window. Kickoff is March 10. Leadership wants to see sequencing against that
window — what lands when — not a date on a slide.
