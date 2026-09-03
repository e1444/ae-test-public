# Summary: Engineering Relay FAQ (living doc)

**Source:** `corpus/02_engineering_relay_faq.md` — wiki page owned by Jonas Meyer, last updated
2025-03-07. Records *decided/verified* facts only; open decisions are explicitly pushed to the PRD.

## Overview
A verified-facts FAQ answering the questions circulating in `#proj-relay`. Distinguishes clearly
between what's confirmed against Stripe docs/AtlasPay integration notes vs. what's still an
unverified assumption or an open design task. Functions as the authoritative "ground truth" on the
Wave 1 mechanics and as a tracker of exactly which Wave 2 questions remain unanswered as of kickoff.

## Stakeholder decisions / verified facts
- **Card import process (step-by-step, verified):** AtlasPay vault export → PGP-encrypt with
  Stripe-issued public key → SFTP upload with manifest (row count + column shape) → Stripe creates
  Customer + PaymentMethod per row (cards only; bank/wallet explicitly out of scope, and Streamhaven
  has none anyway) → Stripe returns a response file, **one row per successfully migrated card only**.
- **No status/error column exists in the response file.** Failures are identified only by diffing
  submitted references against returned rows. Test batch: 15 submitted → 12 returned; 3 absent (2
  expired, 1 bank-closed) — cause known only via cross-checking AtlasPay's own records, not Stripe.
- Card data in the repo is fictional (Stripe test PANs) but must be *handled* as if live: encrypted
  SFTP only, never email/ticket/laptop.
- Not using Stripe's official migration service is final, decided at exec level — not up for debate.
- The in-house dunning engine is being retired outright, regardless of what's decided for the 6
  mid-cycle customers — whatever the end state, it must run on Stripe's side.

## What must stay the same (per this doc)
- Nothing new here beyond prior docs — reinforces renewal-date integrity and zero double-billing
  expectations by cross-reference.

## What must change
- Card data custody moves from AtlasPay tokens to Stripe payment methods via the one-time SFTP
  import (Wave 1) — no reversible/retry path from Stripe's side; failures must be caught by
  Streamhaven's own diffing logic.
- Dunning mechanism moves entirely to Stripe's retry/dunning tooling post cutover (mechanism for the
  6 currently-mid-cycle customers still unresolved).

## Explicitly marked "open / unresolved" by this FAQ (PRD must decide)
1. **Mid-dunning import** — working assumption is Stripe cannot represent "retry 3 of 4 of a 30-day
   cycle," but this is *not yet verified against Stripe docs*. Someone must confirm. Separately,
   what happens to the 6 affected customers is an open decision either way.
2. **Scheduled downgrades** — no defined Stripe-side design yet for the 5 queued pro→basic
   customers (all effective April 2025). Must exist before cutover.
3. **Coupon/discount carry-over** — unsolved; `discounted_periods_remaining` gives exact amounts
   owed per subscriber, but the transfer mechanism into Stripe is undesigned.
4. **"Live on Stripe" definition** — not yet defined; must give ops per-subscription visibility and
   guarantee zero double-billing.
5. **Webhooks / Stripe→internal data sync** — mechanism unresolved; requirement (ops visibility +
   Wren's warehouse feed) is settled, the "how" is not.
6. **Tax approach** — open, owned by Finance; must work before the *first* Stripe invoice, not be
   retrofitted; must handle EU business VAT IDs and flagged tax-exempt accounts.

## Deadline context
- Kickoff: March 10, 2025.
- June 1–15 renewal wave (large cluster of annuals) is the hard backstop — Wave 2 must be complete
  and soaked before it.
