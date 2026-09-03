# Email: Introducing Project Relay

```text
From:     Dana Whitfield <dana.whitfield@streamhaven.example.com>
To:       All Staff
Date:     Mon, 24 Feb 2025, 4:32 PM
Subject:  Project Relay - moving our billing to Stripe
```

Team,

Short version: we are moving Streamhaven's billing to **Stripe**, and the project starts next week.
It has a codename now — **Project Relay** — because that is exactly what it is: a clean handoff of
every active subscription from our in-house engine to Stripe, with nobody's card charged twice and
nobody's discount lost.

**Why now.** We built our billing engine in 2005 and it has been carried by heroes ever since. Every
new payment feature is a quarter of engineering we don't have. We can't do intelligent retries, we can't
scale to new markets cleanly, and our authorization rates trail industry benchmarks. Meanwhile Stripe
gives us mature tooling, international tax support, and a data pipeline our reporting team has been
asking for since 2023.

**Decisions already made** (don't re-open these):

1. **We are not using Stripe's managed migration service.** Their team is great, but we need full
   control over timing and repeatability, and we want the tooling in-house. Engineering is writing
   our own migration scripts. This is final.
2. **Two waves.** First customers and their payment methods. Then billing: plans, prices, coupons,
   subscriptions. Billing cutover completes **before the June renewal wave** — that's the hard
   deadline.
3. **We run this in USD, CAD, and EUR**, exactly as today. No new markets mid-project.
4. Nothing customer-facing changes at cutover except what must change. Support and Marketing will
   drive the communications plan.

What I need from everyone: if your team owns data that touches billing — reporting, marketing
campaigns, support macros, finance reconciliation — your requirements need to be in the room **now**,
not in April. Engineering is running the design review this week; see the calendar invite.

This team has shipped harder things with less warning. Let's go.

Dana

```text
From:     Priya Nair <priya.nair@streamhaven.example.com>
To:       Dana Whitfield; All Staff
Date:     Mon, 24 Feb 2025, 5:04 PM
Subject:  Re: Project Relay - moving our billing to Stripe
```

Adding the finance constraints so they're on record from day one:

- **Zero tolerance for double-billing.** One customer, one charge per period, full stop.
- Revenue recognition must survive. Every migrated subscription keeps its renewal date, price, and
  discount state. I will reconcile the first two months to the penny.
- We will settle our tax approach during Wave 2 planning — open for now, but it must be *decided*
  before any invoice is created in Stripe.
- Auth rate is a board-level metric for us (91.4% last quarter). I expect a written plan for how we
  protect it through the transition.

Priya Nair — CFO
