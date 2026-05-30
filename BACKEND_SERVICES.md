# Backend Services Reference

Servicas runs seven Spring Boot services on Cloud Run, one Postgres database each.
This is the consolidated reference. For system topology see [ARCHITECTURE.md](./ARCHITECTURE.md).
For cross-cutting flows see [KEY_FLOWS.md](./KEY_FLOWS.md).

| Service                       | Owns                                         | Default port |
|-------------------------------|----------------------------------------------|--------------|
| identity-service              | auth, users, roles, feature flags            | 8082         |
| customer-service              | customer-specific data (addresses, prefs)    | 8083         |
| service-marketplace-service   | catalog, providers, bookings, reviews, geo   | 8084         |
| support-case-service          | tickets, disputes, evidence                  | 8085         |
| payment-service               | invoices, charges, refunds, payouts, providers | 8086       |
| notification-service          | push, email, SMS, message threads            | 8087         |
| ai-assistant-service          | AI matching, translation, vision, search     | 8088         |

---

## identity-service

**Purpose** — authentication root and identity store. Issues all JWTs; owns the user,
role, and feature-flag tables; brokers the provider-application intake.

**Controllers**

| Class                        | Base path                              |
|------------------------------|----------------------------------------|
| `AuthController`             | `/api/v1/auth`                         |
| `AdminUserController`        | `/api/v1/admin`                        |
| `ProviderSelfController`     | `/api/v1/provider/me`                  |
| `CustomRoleController`       | `/api/v1/admin`                        |
| `FeatureFlagController`      | `/api/v1` (+ `/feature-flags/public`)  |
| `EmailStatusController`      | `/api/v1/admin/email`                  |

**Entities** — `UserAccountEntity`, `ProviderApplicationEntity`,
`PasswordResetTokenEntity`, `CustomRoleEntity`, `FeatureFlagEntity`, `AuditEventEntity`.

**Lifecycles** — provider application: `Pending → Approved | Rejected`. A new
application is blocked if the user already has one in `Pending`.

**Cross-service** — none outbound. All other services call it indirectly by verifying
the JWT it signed.

**External** — none.

---

## customer-service

**Purpose** — customer-scoped data that isn't part of the marketplace itself: saved
addresses (with geo + label), preferences, favorite providers, waitlist signups,
maintenance reminders, tip records, uploaded media.

**Controllers** — single `CustomerController` at `/api/v1/customer` exposing bundle
endpoints (`/profile`, `/addresses`, `/favorites`, `/reminders`, `/preferences`).

**Entities** — `CustomerAddressEntity`, `CustomerPreferencesEntity`,
`FavoriteProviderEntity`, `MaintenanceReminderEntity`, `WaitlistEntryEntity`,
`TipRecordEntity`, `CustomerMediaEntity`.

**Cross-service / external** — none.

---

## service-marketplace-service

**Purpose** — the catalog, the provider directory, every booking, the canonical
booking state machine, reviews, verification badges, and the geo/regional configuration
(countries, states, cities, pricing/insurance/SLA rules per market).

**Controllers**

| Class                                    | Base path                                |
|------------------------------------------|------------------------------------------|
| `MarketplaceController`                  | `/api/v1/marketplace`                    |
| `ServiceCategoryController`              | `/api/v1/marketplace/admin/categories`   |
| `GeoController`                          | `/api/v1/marketplace/admin/geo`          |
| `ProviderVerificationBadgeController`    | `/api/v1/marketplace`                    |

**Entities (selected)** — `BookingEntity`, `BookingTimelineEventEntity`,
`ServiceCatalogEntity`, `ServiceCategoryEntity`, `ProviderEntity`,
`ProviderSettingsEntity`, `ProviderDocumentEntity`, `ProviderRateEntity`,
`ProviderVerificationBadgeEntity`, `ReviewEntity`, `RegionalPlanEntity`,
`CountryConfigEntity`, `StateEntity`, `CityEntity`, `HolidayEntity`,
`PricingRuleEntity`, `CancellationPolicyEntity`, `InsuranceRequirementEntity`,
`SlaPolicyEntity`, `ServiceTaxRuleEntity`, `QuoteTemplateEntity`,
`InvoiceTemplateEntity`, `ManagedServiceLifecycleEventEntity`,
`FeatureOverrideEntity`.

**Lifecycles**

- **Booking** — canonical 7-status flow: `Requested → Scheduled → Confirmed → OnTheWay
  → Paid → Complete`. Side branches: `Cancelled` (customer or admin) and `Rejected`
  (admin on behalf of an external provider). See
  [KEY_FLOWS.md#booking-state-machine](./KEY_FLOWS.md#booking-state-machine) for full
  transition matrix and role gating.
- **Managed service** — append-only event table (`managed_service_lifecycle_events`)
  recording every state change with `from`, `to`, `actor`, `role`, `reason`.

**Cross-service** — none outbound. **Receives** the `Paid` transition write from
payment-service.

**External** — none.

---

## support-case-service

**Purpose** — support tickets (refund disputes, safety concerns, agent escalations),
evidence attachments, booking-cancellation investigations, agent KPIs.

**Controllers**

| Class                             | Base path                                            |
|-----------------------------------|------------------------------------------------------|
| `TicketController`                | `/api/v1/support`                                    |
| `BookingCancellationController`   | `/api/v1/support/agent/cancellations`                |
| `EvidenceAttachmentController`    | `/api/v1/support/agent/tickets/{id}/evidence`        |
| `SupportMetricsController`        | `/api/v1/support/agent/metrics`                      |

**Entities** — `TicketEntity`, `TicketEventEntity`, `EvidenceAttachmentEntity`,
`BookingCancellationViewEntity`.

**Lifecycles** — ticket event types: `Created`, `StatusChange`, `OwnerChange`,
`PriorityChange`, `Note`, `Safety`, `Refund`, `Comment`. Tickets are append-only via
events; the current state is derived from the latest event.

**Cross-service** — `SupportAiAssistant` → ai-assistant-service for ticket
summarization and suggested-reply generation.

**External** — none.

---

## payment-service

**Purpose** — payment orchestration behind a single internal `PaymentProvider` contract:
invoices, charges, refunds, tips, payout methods, FX rates, and **admin-configured,
runtime-extensible payment providers**.

**Controllers**

| Class                            | Base path                                |
|----------------------------------|------------------------------------------|
| `PaymentController`              | `/api/v1/payments`                       |
| `ProviderConfigAdminController`  | `/api/v1/payments/admin/providers`       |
| `PaymentWebhookController`       | `/api/v1/payments/webhooks/{code}` (permitAll, signature-verified) |
| `FxRateController`               | `/api/v1/payments/admin/fx-rates`        |

**Entities** — `InvoiceEntity`, `PaymentChargeEntity`, `CustomerPaymentMethodEntity`,
`PayoutMethodEntity`, `FxRateEntity`, `PaymentProviderConfigEntity` (code, kind, mode,
enabled, markets/currencies, AES-encrypted credentials + webhook secret).

**Provider model** — an admin can add **any payment type at runtime** (no redeploy) from
the *Payment providers* admin tab. Resolution precedence is *enabled DB config → static
env-gated bean → pending stub*; `PaymentProviderFactory` builds the adapter per
`ProviderKind`:

- First-class: `STRIPE` (PaymentIntents), `PAYPAL` (Orders v2), `SQUARE` (REST; also Cash
  App Pay + wallet methods), `ADYEN` (Checkout, raw HTTP).
- Wallets: `GOOGLE_PAY` / `APPLE_PAY` delegate to a configured card processor.
- Catch-all: `REST_TEMPLATE` (config-driven HTTP — Paystack/Flutterwave/MoMo, etc.) and
  `MANUAL` (offline settlement — bank transfer/Zelle/mobile money; charge stays `PENDING`
  until a webhook or admin settles it). `BRAINTREE` is a typed stub pending its adapter.

Credentials are encrypted at rest (`SecretCipher`, AES-256-GCM, key from
`SERVICAS_PAYMENT_CONFIG_ENCRYPTION_KEY`) and never returned by the API (the admin view
lists only which credential keys are set). The internal model normalizes provider statuses
(e.g. Square `COMPLETED` → `SUCCEEDED`, `APPROVED` → `PENDING`). Charge tokenization happens
client-side via each provider's SDK (see [PANELS.md](./PANELS.md#1-customer-panel)).

**Webhooks** — `POST /api/v1/payments/webhooks/{code}` verifies the provider signature
(Stripe `constructEvent`, Square HMAC, generic shared-secret) and idempotently transitions
the matching charge `PENDING → SUCCEEDED/FAILED/REFUNDED`, flipping the invoice to `Paid`
and notifying marketplace on success.

**Cross-service**

- `MarketplaceBookingClient` → service-marketplace-service: writes `status=Paid` on a
  successful charge (or webhook settlement).
- `NotificationClient` → notification-service: emits `payment.failed` (with retry link)
  and `payment.succeeded`.

**Pending state** — when no real provider is configured the `PendingPaymentProvider` stub
(np/dev only, opt-in) returns an honest `UNCONFIGURED` result rather than faking success.
Per memory rule it is the **only** sanctioned UI fallback in the platform.

---

## notification-service

**Purpose** — outbound communication: push, email, SMS, in-app message threads,
templated event notifications. Owns the user's device-registration list.

**Controllers** — single `NotificationController` at `/api/v1/notifications` with
sub-routes for messages, templates, and channel-specific endpoints (e.g.
`/notifications/payments` for the payment-service callback).

**Entities** — `NotificationEntity`, `NotificationTemplateEntity`,
`MessageThreadEntity`, `MessageEntity`, `PushDeviceEntity`.

**Cross-service** — `AiTranslationClient` → ai-assistant-service for on-the-fly
translation of message bodies.

**External** — FCM / APNs for push, SMTP / SendGrid for email, Twilio (or stub) for SMS.
Specific provider is selected by `SERVICAS_NOTIFICATION_PROVIDER` env.

---

## ai-assistant-service

**Purpose** — every AI-mediated capability in the platform: customer↔provider matching,
ticket summarization, message translation, vision (photo + video) analysis, voice
transcription, business search.

**Controllers**

| Class                       | Base path                              |
|-----------------------------|----------------------------------------|
| `AiAssistantController`     | `/api/v1/ai`                           |
| `AdminAiPromptController`   | `/api/v1/admin/ai-prompts`             |

**Endpoints (selected)** — `POST /ai/translate`, `POST /ai/support/summarize`,
`POST /ai/match/providers`, `POST /ai/vision/analyze`, `POST /ai/transcribe`,
`GET /ai/geocode`, `GET /ai/geocode/reverse`, `POST /ai/business/search`.

**Entities** — `AiPromptOverrideEntity` (admin-tuned system prompts; falls back to
hard-coded defaults when no override is set).

**External** — OpenAI (chat completions, vision, Whisper), Google Places, Google
Geocoding.

**Cross-service** — none outbound; **receives** calls from notification-service
(translation) and support-case-service (summarization).

---

## Deployment conventions

All seven services share the same skeleton:

- `Dockerfile` at repo root, multi-stage build, JDK 24 runtime image (Spring Boot 4.0.x).
- `.github/workflows/ci-cd.yml` — same shape per repo:
  - PR to `main` → test + build + deploy to **np**
  - Merge to `main` → test + build only
  - `workflow_dispatch` → manual prod deploy
- `application-{local,np,postgres,prod}.yml` profiles. Local + tests use H2 in-memory
  on the `local` profile; np uses Neon Postgres; prod uses Cloud SQL.
- All services run `JwtAuthenticationFilter` and share a `common/` package with
  `ApiResponse`, `SecurityConfig`, `JwtService`.
- Configuration is environment-driven: secrets via Google Secret Manager,
  per-service base URLs via `SERVICAS_<SERVICE>_BASE_URL`.

See each repo's own `README.md` for service-specific operational notes.
