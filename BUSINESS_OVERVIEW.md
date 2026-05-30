# Servicas — Business Overview

A pitch-ready brief for investors, advisors, and early hires. Pairs with the
technical [ARCHITECTURE.md](./ARCHITECTURE.md). For day-to-day product surface area
see [PANELS.md](./PANELS.md).

> **Stage disclosure.** Servicas is in pilot. The product is feature-complete across
> seven backend services and five user panels, deployed on Cloud Run (non-prod).
> Live customer traction is pre-launch. Everything below frames where the
> platform is going, not metrics we already have.

---

## 1. Executive summary

**The core idea: centralize the services you need — at home, at the office, at work,
anywhere — into one application, with AI assistance.** Today there is no single place to
get any service you need wherever you need it. Every service lives in its own separate app
(one app for cleaning, another for plumbing, another for lawn or grounds, another for
handyman or IT/office support…), each with its own signup, its own payment, and its own
quality bar. Servicas brings all of them together: one app where a customer can find and
book **any** service — for a home, an office, or a workplace — and an AI assistant guides
them from "this needs fixing" to the right vetted provider, price, and booking.

Servicas is an AI-augmented services marketplace. Customers book vetted providers for
cleaning, HVAC, plumbing, electrical, lawn & grounds, security, office/IT support,
childcare, and emergency repair — across homes, offices, and workplaces, all from one app;
providers run their entire job operations inside it; admins curate the catalog per market;
and regional managers grow new geographies.

**One sentence:** _Servicas is the one app for every service you need — at home, at the
office, or at work — a single, AI-assisted place to get anything done, multi-market,
multi-language, multi-currency, and mobile-first._

**One line on the model:** _take rate on every transaction, plus subscription tiers
for premium provider visibility and verification._

---

## 2. The problem

**Services are fragmented across dozens of apps — there is no central way to get any
service you need, whether at home, at the office, or at work.** As of today a customer
juggles a different app, account, and checkout for each need: one for cleaning, another
for plumbing, another for lawn or grounds, another for handyman, HVAC, or office/IT
support, and informal channels (WhatsApp, Facebook groups, word of mouth) for everything
else. Nobody offers a single place where you can simply ask for *any* service — wherever
you need it — and get matched, priced, and booked. Servicas exists to be that one app,
with an AI assistant that figures out which service you actually need.

On top of that fragmentation, the existing players are weak. Local on-site services are a
$500B+ category globally and still mostly analog. The incumbents (Thumbtack, Angi,
TaskRabbit, Handy) sell **leads, not outcomes** — providers pay per quote even if the
customer never books, and customers wade through unverified bids.

Pain on the customer side
- No way to confirm who actually shows up — verification is opaque.
- Quotes range 3–5× for the same job.
- No standardized recourse for refunds or no-shows; disputes happen in DMs.
- US-centric platforms; people in non-English markets fall back to WhatsApp groups.

Pain on the provider side
- Pay-per-lead is brutal: providers burn $30–$120 to acquire one booking, win-rate ~10%.
- No operating system: jobs are tracked in spreadsheets, payments in Zelle, photos in
  camera roll.
- Single-language platforms exclude immigrant-owned small businesses that
  dominate the labor pool in this category.

Pain on the operator side (admins, regional managers)
- Catalog launch is a content + compliance project (insurance, licensing, holidays,
  tax). Existing platforms expose almost no admin tooling to scale a new market.

---

## 3. The solution

```
   ┌──────────────────────────────────────────────────────────────┐
   │   Customer panel        Provider panel       Admin / Support  │
   │   ─────────────        ──────────────       ────────────────  │
   │   Search · Book        Inbox · Quote        Markets · Catalog │
   │   Pay · Review         Travel · Invoice     Approvals · Audit │
   │   AI-match providers   Customer chat        AI prompt tuning  │
   └────────────────────────────────────────────────────────────────┘
                                  │
                  ┌───────────────┼────────────────┐
                  │  7 backend services on Cloud   │
                  │  Run · per-service Postgres    │
                  │  Single React codebase wraps   │
                  │  web · iOS · Android · desktop │
                  └────────────────────────────────┘
```

What sets Servicas apart from a bookings app
- **One app for every service:** A single place to request *any* service — home, office,
  or workplace — instead of a different app per category. The AI assistant turns a plain
  description ("the AC is leaking", "we need weekly office cleaning") into the right
  category, provider, and price.
- **End-to-end:** From discovery → booking → real-time chat → payment → review →
  dispute resolution. No off-platform handoff.
- **AI-native:** Matching, multi-language message translation, photo/video damage
  assessment, voice transcription, ticket summarization — all built into the
  ai-assistant service, not bolted on.
- **Multi-market from day one:** Countries, states, cities, holidays, pricing rules,
  tax, insurance requirements, and SLA policies are all configurable per region.
- **Operator-grade admin:** Eleven-tab admin workspace (dashboard, bookings, customers,
  providers, services, markets, AI insights, finance, plus runtime controls for customer
  features, provider features, and payment providers) — the kind of back-office
  competitors keep internal-only.
- **One codebase, three platforms:** Capacitor wraps the React bundle as iOS /
  Android; Tauri wraps the same bundle as macOS / Windows / Linux desktop. No
  divergent native teams.

---

## 4. Why now

| Tailwind                                          | Why it matters for Servicas                                  |
|---------------------------------------------------|--------------------------------------------------------------|
| Multimodal AI is finally cheap                    | Per-message translation, photo triage, voice intake are now viable unit-economically. |
| Mobile-first immigrant workforce is underserved   | English-only platforms leave money on the table in TX, FL, CA, NY, AZ, GA. |
| Customer trust in legacy marketplaces is eroding  | TaskRabbit/Angi NPS publicly weak; provider churn rising. |
| Post-COVID home-improvement spend is sticky       | US home-services market grew from ~$430B (2020) to projected $600B+ (2027). |
| Cloud Run / managed Postgres collapsed infra cost | A 7-service stack can run for low-three-digit USD/month on pilot infra. |

---

## 5. Market opportunity

Framing (figures from public industry reports, not Servicas-specific):

| Layer | What it counts                                              | Scale (USD)  |
|-------|-------------------------------------------------------------|--------------|
| **TAM** | Global home / local services GMV                          | ~$1.5T       |
| **SAM** | English + Spanish North America home services GMV         | ~$200B       |
| **SOM** | Reachable in years 1–3 (5 metros, 6 categories, 5% share)  | ~$300M GMV   |

A 12% blended take rate on the SOM is a ~$36M revenue path within three years if
execution lands two metros + Texas-wide coverage. Conservative; comparables show
that the bottleneck is provider supply, not customer demand.

---

## 6. Target customers

### Customer side (demand)

| Segment                  | Why they pick Servicas                                           |
|--------------------------|------------------------------------------------------------------|
| Homeowners 30–55         | Verified providers, transparent pricing, in-app warranty / refund. |
| Bilingual households     | Real-time chat translation removes the language barrier.           |
| Property managers / Airbnb hosts | Recurring scheduling, multi-property dashboards, single invoice across units. |
| Small-business operators | Same flows for office cleaning, AC repair, generator service.      |

### Provider side (supply)

| Segment                          | Why they pick Servicas                                           |
|----------------------------------|------------------------------------------------------------------|
| Independent tradespeople (1–5 staff) | Free job inbox, no pay-per-lead, low take rate vs. Angi (~30%). |
| Immigrant-owned small businesses     | Native-language customer chat + voice intake.                  |
| Established multi-truck shops        | Document / insurance management, regional dispatch.            |

### Operator side (B2B / white-label)

- **Regional managers:** RM workspace gives readiness scoring + recruiting signals to
  launch markets we don't operate directly.
- **Future white-label:** Co-ops, franchisors, and municipal home-aid programs can
  license the admin stack.

---

## 7. Product surface today

Seven Spring Boot services + one React codebase. Detail in
[BACKEND_SERVICES.md](./BACKEND_SERVICES.md); summary here.

```
   identity     →  auth, roles, feature flags
   customer     →  saved homes, prefs, favorites, reminders
   marketplace  →  catalog, providers, bookings, reviews, geo
   payment      →  admin-configured gateways (Stripe · PayPal · Square · Adyen ·
                   Google/Apple Pay · generic) + webhooks; secrets encrypted
   notification →  push · email · SMS · in-app threads
   support      →  tickets · disputes · evidence · agent KPIs
   ai-assistant →  matching · translation · vision · transcription · search
```

Five user panels (Customer, Provider, Admin, Support, Regional) — all shipped as
the same React bundle for web, mobile, desktop.

---

## 8. How Servicas makes money

```
                            ┌────────────────────────────┐
                            │     Revenue streams        │
                            └─────────────┬──────────────┘
        ┌─────────────────┬───────────────┼─────────────────┬───────────────┐
        ▼                 ▼               ▼                 ▼               ▼
  Transaction fee   Provider sub      Featured        FX margin       Insurance /
  (% of booking)    (monthly tiers)   placement       on cross-       warranty
                                      (per slot)      currency        add-ons
```

### 8.1 Transaction take rate (primary)
- 10–15% of GMV per completed booking, blended across categories.
- Charged to the provider on settlement. Customer-facing price stays clean.
- Higher take on low-ticket / one-off (cleaning) where CAC parity drives margin;
  lower take on big-ticket (HVAC install) to keep providers loyal.

### 8.2 Provider subscription tiers
| Tier        | Monthly | Includes                                                       |
|-------------|---------|----------------------------------------------------------------|
| Free        | $0      | Inbox, basic profile, 5% lower take rate than pay-per-lead competitors |
| Pro         | $49     | Verified badge surface, photo gallery, scheduling automation   |
| Business    | $149    | Multi-staff, payroll exports, branded invoices, priority support |
| Fleet       | custom  | API access, regional dispatch, route optimization              |

### 8.3 Featured placement
- Sponsored top-of-list per category × region. Auction or flat-rate per slot.
- Capped per category to preserve customer trust (no more than 1 in 3 results).

### 8.4 Payment & FX margin
- 0.5–1.0% margin on cross-border or multi-currency settlement (FX rate table in
  `payment-service`).
- Pass-through on Stripe/Square fees, no markup on domestic card processing.

### 8.5 Add-on attach (insurance, warranty, tips)
- Optional damage/insurance add-on at checkout, ~5–10% of booking, 20% margin.
- Tips: pass 100% to provider; counts toward retention, not revenue.

### Blended unit economics (illustrative — model, not actual)
```
   Avg booking value (ABV)         $180
   Take rate                       12%        ─►  Gross revenue / booking   $21.60
   Payment processing               ─$5.40
   Notification + AI               ─$0.20
   Support reserve (1%)            ─$1.80
                                              ─►  Contribution / booking    $14.20
   Repeat rate (yr 1)              35%
   LTV / customer (24-month)       $98
   Provider CAC target             $80        ─►  LTV : CAC                 1.2x → 3x at scale
```

---

## 9. Go-to-market

### Phase 1 — single metro pilot (months 0–6)
- Pick one US Sun-Belt metro (Atlanta or Austin — already in the seeder).
- Recruit 100 verified providers across 4 categories.
- Target 1,000 paying customers via paid social + Spanish-language radio.

### Phase 2 — adjacent expansion (months 6–18)
- Add 2 metros in the same state, ride compliance + tax sameness.
- Layer Pro/Business subscriptions on top of free tier.
- Open Regional Manager program for franchise-style market launches.

### Phase 3 — multi-state + multi-language (months 18–36)
- Texas-wide → Florida → California (Spanish-bilingual share is highest there).
- White-label pilot with one co-op or municipal aid program.
- Open API for property-management software integrations.

### Customer acquisition mix
| Channel                | Notes                                                          |
|------------------------|----------------------------------------------------------------|
| Paid social (Meta, TikTok) | Targeted by ZIP + homeowner intent signals                |
| SEO / local pages      | Auto-generated per service × region (marketplace catalog drives content) |
| Spanish-language radio + community partners | Channel underused by incumbents             |
| Provider referral      | Provider invites customer once; both get credit toward next booking |
| Property manager + Airbnb host integrations | High-velocity B2B referrer                  |

### Provider acquisition mix
| Channel                | Notes                                                          |
|------------------------|----------------------------------------------------------------|
| In-person door-knock   | Highest conversion in pilot phase (TaskRabbit's old playbook)  |
| Trade association partnerships | NACE, PHCC, NARI — discount on Pro tier in year one     |
| Existing-platform recruitment | Direct cold to top-rated profiles on competitors        |
| Referral bounty        | $50 per onboarded provider who completes first paid job        |

---

## 10. Competitive landscape

```
                                                  Operator depth (admin / multi-market)
                                                                ▲
                                                                │
                                                          Servicas
                                                                │
                                                                │
              Mr. Handy                                          │
                  ──────────                                    │
                   Booksy                                       │
                                                                │
                                                                │
   Handy / TaskRabbit  ─────────  Thumbtack  ─────────  Angi   │
                                                                │
                                                                │
                                                                │
                                                                ▼
   ◄───────────────────────────────────────────────────────────►
   Lead-based / English-only                AI-native / multi-language / multi-market
```

| Competitor   | Model                | Strength                          | Why Servicas wins                                                                 |
|--------------|----------------------|-----------------------------------|------------------------------------------------------------------------------------|
| Thumbtack    | Pay-per-quote        | Brand, SEO                        | Provider economics broken (sub-10% close rate); we charge on outcome, not lead.    |
| Angi / HomeAdvisor | Lead-gen + Angi Key (subscription) | Massive ad spend       | Customer NPS weak; AI matching gives a more relevant first impression.             |
| TaskRabbit   | Hourly tasker labor  | Brand, IKEA partnership           | Verified provider businesses (not gig taskers) for trades that need licensing.     |
| Handy (ANGI) | Hourly cleaning      | Bundled with Angi                 | Multi-category from day one; not boxed into cleaning.                              |
| Booksy       | Beauty / personal    | Strong vertical playbook          | We pick a different vertical (home) with a 4× larger TAM.                          |
| Local FB groups / WhatsApp | Free informal | Trusted relationships          | We ship the trust layer (verification, escrow, dispute resolution) on top.         |

---

## 11. Moat

1. **Multi-tenant geo + compliance engine.** Country / state / city / holiday /
   tax / insurance / SLA configuration that took 18 months to build correctly. New
   markets launch in days, not quarters.
2. **AI-prompt + model abstraction.** Admin-tuned system prompts per use case
   (matching, summarization, vision) live in DB — not in code. Model swaps don't
   need a deploy.
3. **Single codebase across web / mobile / desktop.** Halves the engineering org
   vs. competitors running parallel iOS + Android + web teams.
4. **Provider-side operating system, not lead-gen.** Once a provider runs their
   entire job flow in Servicas (invoices, photos, payouts, ratings, payroll),
   switching cost is high.
5. **Two-sided liquidity in immigrant + bilingual markets.** Hard for
   English-only incumbents to copy; takes localized GTM + product, not just a
   translate button.

---

## 12. Current state (honest)

| Pillar                    | Status                                                                  |
|---------------------------|-------------------------------------------------------------------------|
| Engineering               | 7 backends + 5 panels live on Cloud Run (non-prod). CI/CD green.       |
| Booking lifecycle         | 7-status canonical state machine, internal + external provider flows.  |
| Payments                  | Admin-configured dynamic providers (Stripe/PayPal/Square/Adyen/wallets/generic), encrypted credentials, webhooks; client-side SDK tokenization. |
| AI                        | Translation, vision, transcription, matching, summarization shipped.    |
| Admin / Regional tooling  | 11-tab admin (incl. feature enablement + payment-provider config) + market readiness scoring. |
| Mobile / desktop          | Capacitor + Tauri builds working; single codebase.                      |
| Customers / GMV           | **Pre-launch.** Pilot infra; no production customer base yet.           |
| Provider supply           | **Pre-launch.** Seeder + onboarding wizard ready; live recruiting next. |
| Compliance                | Documents review flow in place; legal entity per market TBD.            |
| Team                      | Founder + technical contributors; first hires TBD.                      |

---

## 13. Roadmap (rolling 24 months)

```
   Q3 '26                Q4 '26              Q1 '27               Q2 '27               H2 '27 → '28
   ──────                ──────              ──────               ──────               ────────────
   First metro live      Pro tier launch     2nd + 3rd metro      White-label pilot    Multi-state
   100 providers         Subscription rev    Texas-wide           Property mgmt API    EU pilot
   1k customers          Featured slots      RM program           Insurance attach     Series A close
   Pilot insurance att.  Provider referral   Voice-first intake   Tip / payout v2      Public catalog SEO
```

---

## 14. Risks

| Risk                                | Mitigation                                                                |
|-------------------------------------|---------------------------------------------------------------------------|
| Two-sided cold start                | Single-metro launch; door-knock supply before paid demand.                |
| Trust / safety incident             | Background-check partner integration; insurance attach; verified badges.  |
| Payment provider concentration       | Provider-agnostic abstraction; can move volume between Stripe/Square in <1 day. |
| AI cost spike                       | Admin prompt overrides + per-tenant rate limits; cheaper models for non-customer-facing tasks. |
| Regulatory (licensing, tax)         | Per-market compliance engine; legal review per state at launch.           |
| Competitor M&A                      | Speed-to-market in underserved metros; bilingual moat is structural.       |
| Founder concentration               | Documentation + CI/CD complete; bus factor mitigation in roadmap.          |

---

## 15. Use of funds (illustrative — $2M seed ask)

| Bucket                        | %     | Notes                                                  |
|-------------------------------|-------|--------------------------------------------------------|
| Provider acquisition (metro 1) | 25%  | Door-knock + bounty + onboarding ops                   |
| Customer acquisition (metro 1) | 25%  | Paid social, Spanish-language community, local SEO    |
| Engineering (2 hires)         | 25%   | Senior backend + mobile lead                           |
| Trust & safety / compliance   | 10%   | Background-check vendor, insurance attach pilot       |
| Infra + AI cost runway        | 5%    | 12-month buffer at projected scale                     |
| Working capital / G&A         | 10%   | Legal, accounting, regional entity setup               |

Burn target: ~$110k/month at month 6, ~$160k/month at month 12.

---

## 16. Why us, why now (closing)

The platform is built. The 7-service backend, the 5-panel React app, the AI layer,
the payment abstraction, the multi-market admin — all shipped and CI-green. What
remains is the GTM motion: pick a metro, recruit 100 providers, acquire the first
1,000 customers, and let the operating-system moat compound from there.

Most marketplaces raise to build the product. We're raising to put a built product
into market.

---

## 17. Contact

- Founder: Olivier Santos
- Repo (read-only investor access on request): github.com/servicas
- This document: [docs/BUSINESS_OVERVIEW.md](./BUSINESS_OVERVIEW.md)

_Document version: pilot · last updated 2026-05-25._
