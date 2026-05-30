# Panel Reference

Servicas exposes five panels, each as a lazy-loaded React workspace under
[src/features/](../src/features/). All five share the same chrome (top nav,
notification bell, profile menu) but render different feature trees driven by
`usePortalSession().portalRoute`.

| Panel    | Root component                                                            | Audience                            |
|----------|---------------------------------------------------------------------------|-------------------------------------|
| Customer | [CustomerWorkspace.tsx](../src/features/customer/CustomerWorkspace.tsx)   | End customers booking services      |
| Provider | [ProviderWorkspace.tsx](../src/features/portals/ProviderWorkspace.tsx)    | Service providers fulfilling jobs   |
| Admin    | [AdminWorkspace.tsx](../src/features/portals/AdminWorkspace.tsx)          | Internal operators / regional admins |
| Support  | [SupportWorkspace.tsx](../src/features/portals/SupportWorkspace.tsx)      | Support agents handling disputes    |
| Regional | [RegionalWorkspace.tsx](../src/features/portals/RegionalWorkspace.tsx)    | Regional managers for market growth |

The shorter overview lives in [PORTAL_WORKFLOWS.md](./PORTAL_WORKFLOWS.md); this
document goes deeper with diagrams and per-section breakdowns.

---

## 1. Customer panel

The customer panel is the booking journey end-to-end: discover → book → communicate →
pay → review.

### Sections

| Section          | Component                                                                       | Purpose                                                       |
|------------------|---------------------------------------------------------------------------------|---------------------------------------------------------------|
| Dashboard        | [CustomerDashboard.tsx](../src/features/customer/CustomerDashboard.tsx)         | Active bookings, reminders, saved homes, recommendations      |
| Booking          | [CustomerBookingWorkspace.tsx](../src/features/booking/CustomerBookingWorkspace.tsx) | Service picker → provider selection → confirmation       |
| Providers hub    | [CustomerProvidersHub.tsx](../src/features/customer/CustomerProvidersHub.tsx)   | Chat threads, favorites, ratings                              |
| Payments hub     | [CustomerPaymentsHub.tsx](../src/features/customer/CustomerPaymentsHub.tsx)     | Invoices, receipts, saved methods, dispute / refund; pay/tip opens [Checkout.tsx](../src/features/payments/Checkout.tsx) which lists the admin-enabled providers and tokenizes via the provider SDK |
| Support center   | [SupportWorkspace embedded view]                                                | Create / view tickets attached to bookings                    |
| Profile          | [ProfileManager.tsx](../src/features/profile/ProfileManager.tsx)                | Identity, phone, addresses, preferences                       |

### Dominant business flow

```
 ① Customer picks saved address
      Frontend  →  customer-svc   :  GET /customer/addresses
      Frontend  →  marketplace    :  GET /marketplace/services?region=…
      marketplace → Frontend      :  available services in region

 ② Customer describes problem + urgency
      Frontend  →  ai-assistant   :  POST /ai/match/providers
      ai-assistant → Frontend     :  ranked provider suggestions

 ③ Customer picks provider + slot
      Frontend  →  marketplace    :  POST /marketplace/bookings
      marketplace → Frontend      :  booking { status: "Scheduled" }
      marketplace → notification  :  push to provider

 ④ Customer pays invoice  (later)
      Frontend  →  payment        :  POST /payments/charges
      payment   →  marketplace    :  PUT /bookings/{id}/status = Paid
      payment   →  notification   :  payment.succeeded
      notification → Customer     :  receipt + push
```

Initial status auto-advances to `Scheduled` when a slot is provided; otherwise it
remains `Requested`. See [KEY_FLOWS.md](./KEY_FLOWS.md#booking-state-machine).

---

## 2. Provider panel

The provider panel is the job operations cockpit. Internal (verified) providers
drive their own state transitions; external (`ai:` / `svc:` prefixed) providers are
driven by admin/support on their behalf.

### Sections

| Section          | Purpose                                                                  |
|------------------|--------------------------------------------------------------------------|
| Jobs inbox       | Bookings in `Requested` and `Scheduled`; accept / decline / quote        |
| Active jobs      | Bookings in `Confirmed`, `OnTheWay`, `Paid`; status transitions          |
| Schedule         | Calendar view of upcoming work                                           |
| Customer chat    | Per-booking message thread (notification-service backed)                 |
| Invoices         | Send invoice for completed work; tip + refund views                      |
| Ratings          | Latest reviews + aggregate                                               |
| Business setup   | [ProviderBusinessWizard.tsx](../src/features/provider/ProviderBusinessWizard.tsx) — identity, phone, address, categories, documents |
| Documents        | License / insurance / certification uploads, expiry warnings             |

### Dominant business flow — job execution

```
   Booking lands in provider inbox at status = Scheduled

 ① Provider confirms job
      Frontend  →  marketplace    :  PUT /bookings/{id}/status = Confirmed
      marketplace → notification  :  booking.confirmed
      notification → Customer     :  "Your provider confirmed"

 ② Provider starts travel
      Frontend  →  marketplace    :  PUT /bookings/{id}/status = OnTheWay

 ③ Work completed on site
      Frontend  →  marketplace    :  PUT /bookings/{id}/status = Paid
                                     (or auto-set by payment-svc on charge)

 ④ Provider closes the job
      Frontend  →  marketplace    :  PUT /bookings/{id}/status = Complete
      marketplace → notification  :  booking.complete
      notification → Customer     :  "Leave a review"

   External providers: every PUT above is driven from admin/support, not
   the provider workspace.
```

For external providers the **same diagram** applies but every `PUT
/bookings/{id}/status` is driven from admin/support, not the provider workspace.

---

## 3. Admin panel

The admin workspace ships with eleven tabs, each consistent with the same
`.admin-page-header` layout (eyebrow + h2 + intro). The tabs share the customer-style
card system to keep visual parity across panels. The last three are runtime-control
tabs added on top of the original eight.

```
                              AdminWorkspace
                                    │
   Dashboard · Bookings · Customers · Providers · Services · Markets · AI Insights · Finance
                                    │
   ├─ Customer features   (toggle customer portal sections + capabilities)
   ├─ Provider features   (toggle provider portal sections + tools)
   └─ Payment providers   (add / configure any payment gateway at runtime)
```

| Tab               | Owns                                                                        | Key endpoint(s)                              |
|-------------------|-----------------------------------------------------------------------------|----------------------------------------------|
| Dashboard         | Single-shot rollup: GMV, bookings/day, provider supply, market readiness    | `GET /marketplace/admin/dashboard`           |
| Bookings          | Cross-customer booking view, force-transition for external providers        | `GET /marketplace/bookings`, `PUT /bookings/{id}/status` |
| Customers         | User search, status (Active/Suspended), audit trail                         | `GET /admin/users`, `POST /admin/users/{id}/status` |
| Providers         | Application queue (Pending/Approved/Rejected), documents review, badges     | `GET /provider/applications`, `POST /provider/applications/{id}/approve` |
| Services          | Service catalog CRUD, category taxonomy, managed-service lifecycle events   | `GET /marketplace/admin/categories`, `POST /marketplace/services` |
| Markets           | Country/state/city configuration, pricing & insurance & SLA rules, holidays | `GET /marketplace/admin/geo`, market-readiness scoring |
| AI Insights       | OpenAI prompt overrides, model usage stats, vision/transcription review     | `GET /admin/ai-prompts`, `PUT /admin/ai-prompts/{key}` |
| Finance           | FX rates, commission, company payout account, revenue analytics             | `GET /payments/admin/fx-rates`               |
| Customer features | Enable/disable each customer portal section + capability flags (media, provider contact/chat) | `GET/PUT/DELETE /api/v1/admin/feature-flags` (identity) |
| Provider features | Enable/disable each provider portal section + optional tools                | `GET/PUT/DELETE /api/v1/admin/feature-flags` (identity) |
| Payment providers | Add/configure any payment gateway (kind, credentials, markets, mode), enable in checkout | `GET/POST/PUT/DELETE /api/v1/payments/admin/providers` |

### Dominant business flow — provider approval

```
 ① Admin opens Providers tab
      Frontend  →  identity       :  GET /admin/provider/applications?status=Pending
      identity  →  Frontend       :  pending application list

 ② Admin reviews documents
      Frontend  →  marketplace    :  GET /provider/documents?applicationId=…
      marketplace → Frontend      :  documents + expiry warnings

 ③ Admin approves with notes
      Frontend  →  identity       :  POST /provider/applications/{id}/approve
      identity  →  marketplace    :  provision ProviderEntity in catalog
      identity  →  notification   :  provider.application.approved
      notification → Provider     :  "You're live — set your service area"
```

---

## 4. Support panel

Support owns disputes, refund requests, cancellation investigations, and safety
escalations. The workspace centers on a single ticket queue with role-based filters.

### Sections

| Section                   | Purpose                                                                  |
|---------------------------|--------------------------------------------------------------------------|
| Ticket queue              | All tickets, filterable by status / priority / owner                     |
| Ticket detail             | Event timeline (`Created`, `StatusChange`, `Note`, `Refund`, …)          |
| Evidence                  | File uploads scoped per ticket (photos, receipts, transcripts)           |
| Booking-cancellation view | Bookings cancelled / rejected with audit context for agent triage        |
| Agent metrics             | KPIs: avg handle time, resolution count, SLA adherence                   |
| AI assist                 | Summarize ticket, suggest reply, language detection                      |

### Dominant business flow — refund dispute

```
 ① Customer requests refund
      Frontend  →  support        :  POST /support/tickets { category: Refund }
      support   →  notification   :  ticket.created  →  support queue

 ② Agent opens ticket
      Frontend  →  support        :  GET /support/tickets/{id}
      support   →  ai-assistant   :  POST /ai/support/summarize
      ai-assistant → support      :  3-sentence brief (logged as Note event)

 ③ Agent approves refund
      Frontend  →  support        :  POST /support/tickets/{id}/refund
      support   →  payment        :  POST /payments/charges/{id}/refund
      payment   →  notification   :  payment.refunded
      notification → Customer     :  "Refund processed"
      support   →  (self)         :  append TicketEvent { type:Refund, status:Resolved }
```

---

## 5. Regional panel

Regional managers ("RM") are scoped to one or more countries/regions. The workspace
is read-mostly with planning tools layered on top of the marketplace's geo data.

### Sections

| Section           | Purpose                                                                       |
|-------------------|-------------------------------------------------------------------------------|
| Market analytics  | Provider supply by category, booking velocity, GMV by region                  |
| Readiness         | Per-country score (0–100) with category/region breakdown                      |
| Recruiting        | Synthetic provider gaps, suggested categories to enable in a region           |
| Regional plan     | Append-only plan entries (target market, target date, owner)                  |
| Coverage map      | Visual heatmap of provider density vs. demand                                 |

### Dominant business flow — market readiness check

```
 ① RM picks country = "US-TX"
      Frontend  →  marketplace    :  GET /marketplace/admin/dashboard?country=US-TX
      marketplace → Frontend      :  rollup { providers:42, gmv:$X, readiness:67 }

 ② RM drills into "Plumbing"
      Frontend  →  marketplace    :  GET /marketplace/providers?category=Plumbing&region=US-TX
      marketplace → Frontend      :  provider list with verified flag + rating

 ③ RM creates plan entry
      Frontend  →  marketplace    :  POST /marketplace/regional-plans
                                     { target:"Recruit 5 plumbers", due:"Q3" }
```

Readiness scoring is computed live in `MarketplaceService.computeProviderSupplyScore`
(category × region → providers count × 12, capped at 100). The RM workspace surfaces
this without persisting it.

---

## Shared chrome

All five panels share:

- **AppChrome** ([src/features/app/AppChrome.tsx](../src/features/app/AppChrome.tsx))
  — top nav, role indicator, environment banner on np.
- **Notification bell** — polls notification-service every 30s; opens drawer with
  unread items + message threads.
- **Profile menu** — switch language, logout, view session info.
- **`portalRoute` guard** — `PortalAuthGateway` ([src/features/auth/PortalAuthGateway.tsx](../src/features/auth/PortalAuthGateway.tsx))
  redirects authenticated users to their role's workspace; unauthenticated users land on
  the auth surface with three modes — **sign in**, **sign up**, **forgot password** — and
  two sign-in methods: **password** or a **one-time code (OTP)** to email or phone. The
  reset flow verifies an OTP, then sets a new password.

Per memory rule: panels show **empty / error states only**; never canned fallback data.
The single exception is the payment-service `PendingPaymentProvider` when no real
provider is configured.
