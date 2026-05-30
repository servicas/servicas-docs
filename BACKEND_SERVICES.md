# Backend Services Reference

Servicas runs seven Spring Boot services on Cloud Run, one Postgres database each.
This is the consolidated reference. For system topology see [ARCHITECTURE.md](./ARCHITECTURE.md).
For cross-cutting flows see [KEY_FLOWS.md](./KEY_FLOWS.md).

| Service                       | Owns                                         | Default port |
|-------------------------------|----------------------------------------------|--------------|
| identity-service              | auth, users, roles, feature flags            | 8081         |
| customer-service              | customer-specific data (addresses, prefs)    | 8082         |
| service-marketplace-service   | catalog, providers, bookings, reviews, geo   | 8083         |
| support-case-service          | tickets, disputes, evidence                  | 8084         |
| payment-service               | invoices, charges, refunds, payouts          | 8085         |
| notification-service          | push, email, SMS, message threads            | 8086         |
| ai-assistant-service          | AI matching, translation, vision, search     | 8087         |

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

**Purpose** — payment orchestration across four providers behind a single internal
contract: invoices, charges, refunds, tips, payout methods.

**Controllers**

| Class                  | Base path                                |
|------------------------|------------------------------------------|
| `PaymentController`    | `/api/v1/payments`                       |
| `FxRateController`     | `/api/v1/payments/admin/fx-rates`        |

**Entities** — `InvoiceEntity`, `PaymentChargeEntity`,
`CustomerPaymentMethodEntity`, `PayoutMethodEntity`, `FxRateEntity`.

**Cross-service**

- `MarketplaceBookingClient` → service-marketplace-service: writes
  `status=Paid` on successful charge.
- `NotificationClient` → notification-service: emits `payment.failed` (with retry
  link) and `payment.succeeded`.

**External** — Stripe (PaymentIntents), Square (REST), PayPal (Orders v2), Apple Pay
(delegated through Stripe). The internal model normalizes provider-specific statuses
(Square `COMPLETED` → `SUCCEEDED`, Square `APPROVED` → `PENDING`, …).

**Pending state** — when no provider is configured the service uses
`PendingPaymentProvider`, which queues the charge so the UI can still complete the
booking flow. Per memory rule, this is the **only** sanctioned UI fallback in the
platform.

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

- `Dockerfile` at repo root, multi-stage build, JRE 17 runtime image.
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
