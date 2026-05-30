# Key Flows

Cross-cutting business flows that span multiple panels and services. For per-panel
behavior see [PANELS.md](./PANELS.md); for service contracts see
[BACKEND_SERVICES.md](./BACKEND_SERVICES.md).

- [Booking state machine](#booking-state-machine)
- [Payment + retry flow](#payment--retry-flow)
- [Provider onboarding](#provider-onboarding)
- [Support ticket lifecycle](#support-ticket-lifecycle)

---

## Booking state machine

The marketplace service owns the canonical booking lifecycle. Seven statuses on the
happy path, plus two terminal side branches.

```
   Happy path  (left to right)

   ┌───────────┐  pick   ┌───────────┐ confirm ┌───────────┐ depart ┌──────────┐  charge  ┌────────┐  close   ┌──────────┐
   │ Requested │────────►│ Scheduled │────────►│ Confirmed │───────►│ OnTheWay │─────────►│  Paid  │─────────►│ Complete │
   └─────┬─────┘  slot   └─────┬─────┘         └─────┬─────┘        └──────────┘   svc    └────────┘  out     └──────────┘
         │                     │                     │                                                            ▲ terminal
         │ customer/admin      │ customer/admin      │ customer/admin
         │ cancel              │ cancel              │ cancel
         │                     │                     │
         └────────┐  ┌─────────┘  ┌──────────────────┘
                  ▼  ▼            ▼
              ┌──────────────────────┐
              │      Cancelled       │  ── terminal
              └──────────────────────┘

         │ admin/support             │ admin/support
         │ reject (external)         │ reject (external)
         ▼                           ▼
   ┌──────────────────────────────────────┐
   │              Rejected                │  ── terminal
   └──────────────────────────────────────┘
```

### Initial status

`createBooking` sets `Requested`. If the request includes both `scheduledDate` and
`scheduledTime`, the test setup auto-advances to `Scheduled` to mirror the typical
client flow (slot picked at creation). External provider bookings (providerId starts
with `ai:` or `svc:`) always start at `Requested` and need admin action.

### Role gating per transition

| Transition                 | Internal provider | External provider     |
|----------------------------|-------------------|------------------------|
| Requested → Scheduled      | provider / admin  | admin / support        |
| Scheduled → Confirmed      | provider / admin  | admin / support        |
| Confirmed → OnTheWay       | provider / admin  | admin / support        |
| OnTheWay → Paid            | payment-svc / admin | payment-svc / admin  |
| Paid → Complete            | provider / admin  | admin / support        |
| * → Cancelled              | customer / admin  | customer / admin       |
| Requested|Scheduled → Rejected | n/a            | admin / support        |

Enforcement lives in `MarketplaceService.validateTransition` — illegal transitions
throw `BookingTransitionException` → HTTP 409. Terminal statuses (`Complete`,
`Cancelled`, `Rejected`) block further transitions.

### Active-booking guard

`createBooking` also blocks duplicate concurrent bookings: if the same customer
already has a non-terminal booking for the same service, a 409 is returned. The
duplicate check matches `provider.categories` against either the service's category
bucket (e.g. "Home Repair") or its name (e.g. "HVAC") — both styles are used in the
codebase.

### Timeline

Every transition writes a `BookingTimelineEventEntity`: `Status: <toStatus>` +
optional detail. The customer/provider/admin UIs render these as a chronological
audit trail per booking.

---

## Payment + retry flow

```
 ① Customer pays invoice
      Frontend  →  payment        :  POST /payments/charges { invoiceId, methodId }
      payment   →  Stripe/Square/PayPal :  create PaymentIntent / order
      provider  →  payment        :  result

   ┌─ on SUCCESS ────────────────────────────────────────────────────────────┐
   │   payment      →  marketplace    :  PUT /bookings/{id}/status = Paid    │
   │   payment      →  notification   :  POST /notifications/payments        │
   │                                     { event: succeeded }                │
   │   notification →  Customer       :  receipt + push                      │
   │   payment      →  Frontend       :  200 { status: SUCCEEDED }           │
   └─────────────────────────────────────────────────────────────────────────┘

   ┌─ on FAILURE ────────────────────────────────────────────────────────────┐
   │   payment      →  notification   :  POST /notifications/payments        │
   │                                     { event: failed, retryUrl }         │
   │   notification →  Customer       :  email + SMS                         │
   │                                     "Your card was declined — retry"    │
   │   payment      →  Frontend       :  402 { status: FAILED, reason }      │
   │   Frontend     →  Customer       :  inline retry CTA                    │
   └─────────────────────────────────────────────────────────────────────────┘
```

### Pending provider exception

If no payment provider is configured (e.g. cold-start np with no Stripe secret),
`PendingPaymentProvider` is selected. It returns `PENDING` immediately so the booking
flow doesn't dead-end; an admin task drains pending charges once a provider is wired
up. This is the only sanctioned "fake success" path in the platform; everywhere else
the rule is *empty / error states only*.

### Provider abstraction

`PaymentService` selects a provider per charge using the customer's saved payment
method and falls back via `SERVICAS_PAYMENT_PROVIDER_ORDER` (e.g.
`stripe,square,paypal`). Status normalization happens in each adapter so the
internal `PaymentChargeEntity.status` is always one of `PENDING | SUCCEEDED | FAILED
| REFUNDED`.

---

## Provider onboarding

End-to-end from application submission to live catalog appearance.

```
 ① User completes the wizard
      Step 1  —  Identity  (name, phone, address autocomplete via Google Places)
      Step 2  —  Categories  (scoped to enabled markets in user's region)
      Step 3  —  Documents  (license, insurance, certifications)

 ② Submit
      Frontend  →  identity       :  POST /provider/me/applications
                                     (multipart: phone, docs)
                                     email is taken from JWT, NOT request body
      identity  →  (self)         :  persist ProviderApplicationEntity (status=Pending)
                                     sync phone → UserAccountEntity if new

   ┌─ already has Pending ─────────────────────────────────────────┐
   │   identity  →  Frontend  :  409 DuplicatePendingApplication   │
   │   Frontend  →  User      :  "You already have an application" │
   └───────────────────────────────────────────────────────────────┘

   ┌─ first application ───────────────────────────────────────────┐
   │   identity  →  Frontend  :  201 created                       │
   └───────────────────────────────────────────────────────────────┘

 ③ Admin review
      Frontend  →  identity       :  GET /admin/provider/applications?status=Pending
      Frontend  →  marketplace    :  GET /provider/documents?applicationId=…
      Frontend  →  identity       :  POST /provider/applications/{id}/approve  (or /reject)
      identity  →  marketplace    :  provision ProviderEntity in catalog
      identity  →  notification   :  provider.application.{approved|rejected}
      notification → User         :  email + push
```

### Identity → catalog sync

On approval, identity-service emits the approved provider into the marketplace
catalog with the application's name, phone, email, address, and selected categories.
The provider then appears in customer search results for matching regions.

### Documents

Uploaded files are streamed to the marketplace's `provider_documents` table via
`marketplaceClient.uploadMyProviderDocument`. The admin "Submitted documents" card
renders kind + size + content-type + uploaded-at + expiry warning (amber ≤30 days,
red if expired) for production-grade review.

---

## Support ticket lifecycle

Tickets are append-only event streams. The current status is the latest
`StatusChange` event.

```
                          ┌──────────┐
                  create ►│   Open   │
                          └─────┬────┘
                  ┌─────────────┼─────────────┐
                  │ awaiting    │ agent picks │
                  │ customer    │ up          │
                  ▼             ▼             │
            ┌──────────┐  ┌────────────┐      │
            │ Pending  │─►│ InProgress │◄─────┘
            └──────────┘  └─────┬──────┘
                                │
                  ┌─────────────┼────────────┐
                  │ resolution  │ above      │
                  │ committed   │ authority  │
                  ▼             ▼            
            ┌──────────┐  ┌────────────┐
            │ Resolved │  │ Escalated  │── senior closes ──┐
            └─────┬────┘  └────────────┘                   │
                  │                                        ▼
       ┌──────────┼──────────┐                       ┌──────────┐
       │ confirms │ disagrees │                      │ Resolved │
       ▼          ▼          ▼                       └────┬─────┘
   ┌────────┐  ┌──────────┐                               │
   │ Closed │  │ Reopened │── re-triaged ──► InProgress   │
   └────────┘  └──────────┘                               │
       ▲ terminal                                         │
       └──────────────────────────────────────────────────┘
```

### Event types

`Created`, `StatusChange`, `OwnerChange`, `PriorityChange`, `Note`, `Safety`,
`Refund`, `Comment`. Each event captures actor, role, timestamp, and a free-text
body. The ticket detail view renders this stream as a vertical timeline.

### AI assist

For any ticket the agent can trigger `POST /ai/support/summarize` (3-sentence brief)
or `POST /ai/support/suggest-reply` (templated draft response). Both write back into
the ticket as `Note` events tagged `actor=AI` so the audit trail is preserved.

### Cross-flow links

- A refund-category ticket can fire `POST /payments/charges/{id}/refund`. The
  resulting `payment.refunded` notification is also written as a `Refund` event on
  the ticket for traceability.
- A safety-category ticket on an active booking can drive a `Cancelled` transition
  on the booking via admin override.
