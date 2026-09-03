# Billing Migration Assessment

**Time limit: 60–90 minutes. One deliverable.**

> **This is not a conventional coding test.** There is nothing to run and no tests to pass. We are
> evaluating how effectively you use AI tools to absorb a messy, realistic stakeholder landscape and
> turn it into a technical plan a team could actually execute. Use any AI tools you have, any number
> of screens, whatever helps. Justify your decisions — there is no single correct answer.

> **Everything in this assessment is fictional.** Streamhaven, AtlasPay, all people, and all records
> are invented. Card numbers are Stripe's public test numbers. No real data is used anywhere.

---

## The situation

**Streamhaven** is a ~20-year-old streaming SaaS (Basic 1080p / Pro 4K-ad-free, plus a legacy
"Founders" plan). It has been running **its own in-house billing engine** and has decided to move to
**Stripe**.

The facts you need up front:

- Streamhaven stores **raw card numbers (PANs)** from signup — it is PCI-scoped — but it does **not**
  bill on raw cards. Each PAN was sent once to **AtlasPay**, Streamhaven's card processor and token
  vault, which returned a **token** used for every subsequent billing run.
- Stripe's card import moves **payment methods only** — the PANs: an encrypted file is uploaded
  over SFTP, Stripe processes it, and returns a **migration response file** mapping old references
  to new Stripe IDs. A sample of what Stripe returns is in `data/`. That's the complete extent of
  Stripe's involvement: the card vault comes across, and **everything else — the billing itself —
  Streamhaven migrates itself.**
- The migration runs in **two waves**: Wave 1 moves customers + payment methods; Wave 2 moves
  billing — plans, prices, coupons, and subscriptions.
- **Stripe offers an official migration service/tool. Streamhaven has decided not to use it** — the
  team is writing its own scripts. That decision is made; don't relitigate it.
- There are **~100 customers** to migrate, currently billed by custom code against internal tables.
- Streamhaven operates in **USD, CAD, and EUR**.
- Leadership wants this **done before the June renewal wave**. Today is **March 10, 2025**.

## What you have

- **`corpus/`** — pre-kickoff conversations and documents from across the company: executive
  announcement, engineering design review and follow-up FAQ, a reporting-team thread, a marketing
  coupon review, a leadership checkpoint, and a support-team discussion. The details that matter are
  in these conversations — nobody will hand you a requirements list.
- **`data/`** — the current internal tables (`customers`, `subscriptions`, `payment_methods`), the
  prepared **Card Vault Export** for Stripe, and a sample **Migration Response File** from a recent
  test batch with Stripe. Schemas and conventions are documented in `data/README.md`.

Read both with your AI tools. Extract what matters. Where documents disagree or stay silent, that is
yours to resolve and justify.

## Your deliverable: `PRD.md`

Write a **technical PRD at the repo root** that lays out everything that has to happen to get
Streamhaven fully onto Stripe — concrete enough that an engineer could start building from it.

It must, at minimum, address these **four pillars**:

1. **Launch strategy** — how users actually go live: batching, ordering, the cutover moment, what
   "live on Stripe" means operationally, and the sequencing that gets it all done before the June
   renewal wave.
2. **Customer & payment-method migration (Wave 1)** — how card data gets from Streamhaven/AtlasPay
   to Stripe, how the response file is consumed, and what happens to the cards that don't make it.
3. **Billing & subscription migration (Wave 2)** — how plans, prices, coupons, discounts, trials,
   dunning states, and scheduled downgrades are recreated in Stripe without a single double-charge
   or lost discount.
4. **Reporting requirements** — how the reporting team keeps its dashboards working during and after
   the move.

Also required:

- **Sample Stripe calls** — the actual JSON payloads you'd send for every object your plan creates
  (customers, payment methods, subscriptions, prices, coupons…), with the non-obvious fields
  called out and why they matter.
- **Database & internal-system changes** — what has to change in Streamhaven's own tables to support
  the migration and to record that a customer is live on Stripe.
- **Customer communications** — what customers hear, when, and what changes for them.
- **Open decisions** — the conversations leave real questions unresolved. Take positions: the
  options as you see them, your recommendation, and who should make the final call.

## Worth pressure-testing

- What has to change in **Streamhaven's own systems** — and how will anyone know a customer is
  live on Stripe?
- Where could this migration quietly bill someone twice — or not at all — and would anyone notice
  in time?
- The pillars above are not a requirements list. **The corpus contains problems nobody will
  summarize for you**, and conversations that end without resolution. Deciding what must be settled
  before launch versus what gets flagged with an owner is part of the job.

## What we're evaluating

- **Signal from noise** — can you pull the load-bearing details out of long conversations?
- **Stripe fluency** — are you using the platform's actual primitives and mechanics correctly?
- **Risk anticipation** — double-billing, lost discounts, auth-rate drops, silent failures.
- **Decision-making under ambiguity** — reasonable calls, clearly justified, consequences owned.
- **Handling open decisions** — options on the table, a recommendation made, a decision-maker
  named.

A complete-but-shallow PRD loses to a shorter one that nails the hard parts. Structure it however
you like. Spend your time where the risk is.
