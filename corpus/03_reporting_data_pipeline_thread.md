# Slack Export: #data-reporting — "Relay: what the warehouse needs"

**Date range:** March 4–6, 2025

---

**wren.tanaka** — Mar 4, 9:12 AM
Team — Project Relay thread. We've been told to choose our Stripe data strategy *now* so the PRD
can include it. Finance dashboards, exec KPIs, churn models — all of it reads from billing today.

**dev.kapoor** — Mar 4, 9:15 AM
we're going with **Stripe Data Pipeline**, right? we discussed in Jan. daily loads to the warehouse,
all core objects: charges, invoices, subscriptions, customers, credit notes, refunds.

**wren.tanaka** — Mar 4, 9:16 AM
Yes. Data Pipeline is decided. The question is what we lose in translation, and my bet is: our
business dimensions. Stripe doesn't know what a "Founders subscriber" means to *us*, or which
campaign brought someone in, or that a Pro-annual in Berlin is the segment the board asks about.

**dev.kapoor** — Mar 4, 9:18 AM
right. stripe gives us their ids and amounts. our joins break unless our ids ride along

**wren.tanaka** — Mar 4, 9:20 AM
Exactly. So here's my working list of what each recurring report needs. Poke holes in it.

**1. MRR & revenue by plan and billing interval**
The baseline board-deck cut: tier, billing period, currency. Today it comes from our tables.

**2. Legacy vs. new-plan revenue split**
The board literally asks "what's Founders revenue this quarter." That dimension has to survive
the move somewhere.

**3. Acquisition cohorts**
Channel = organic / paid_search / referral / partner. Lives in our `customers` table today. If it
doesn't survive the move, our CAC-payback dashboards die at cutover.

**4. Discount tracking — the finicky one**
Finance wants: which coupons are live, redemption counts, *and* how far through a discount window
each subscriber is (START25 winds down month by month; we forecast with that). We know it per
subscription today. It has to keep flowing after the move.

**5. Churn & dunning recovery**
After the move, failed-payment recovery runs on Stripe's side. I still need: failed payment
counts, recovery outcomes, days past due. Whatever Stripe calls those things, I need them in the
warehouse.

**6. Region / currency rollups**
We report USD, CAD, EUR separately and EU vs NA. Country and currency already exist in our data;
they need to survive the move.

**7. The audit join**
Every Stripe object we create should be joinable back to `CUST-####` / `SUB-####` in our warehouse.
Non-negotiable. Priya reconciles the first two months to the penny.

**dev.kapoor** — Mar 4, 9:34 AM
so the real question is which of these dimensions travel inside Stripe versus live only in our
warehouse — and how. whatever we choose is a one-way door: we can't reshape it after 100
subscriptions exist

**wren.tanaka** — Mar 4, 9:36 AM
Exactly. And that's why this thread exists before Wave 2, not after. Retrofitting means touching
every object again.

**mike.from.finance** — Mar 5, 8:02 AM
Reading along. Two asks from finance:
1. Statement descriptor changes mid-project — flag us, revenue ops gets false-fraud reports when
   descriptors change.
2. When Data Pipeline goes live, we still need month-end numbers to tie out to the penny for the
   first two months post-cutover. Whatever "source of truth" means at that point, decide it and
   write it down.

**wren.tanaka** — Mar 5, 8:10 AM
Both fair. Descriptor decision is with support/eng thread.

**dev.kapoor** — Mar 6, 5:47 PM
last thing before the weekend: the migration itself needs a backfill story. wave 2 creates 100
subscriptions in stripe — do historical invoices get re-created too? we don't want the revenue
dashboards to show a cliff at cutover.

**wren.tanaka** — Mar 6, 5:52 PM
Good catch, that's a real PRD question: what history moves to Stripe, what stays in our warehouse,
and where do the dashboards read during the overlap? Add it to the open questions.
