# Servicas — System Architecture

Servicas is a multi-panel home-services marketplace: customers book providers, providers
operate jobs, admins curate the catalog and markets, and support handles disputes. The
platform ships as a single React codebase wrapped for web / mobile (Capacitor) / desktop
(Tauri), backed by seven Spring Boot microservices on Google Cloud Run.

This document is the entry point. Drill-down docs:

- [BACKEND_SERVICES.md](./BACKEND_SERVICES.md) — per-service reference
- [PANELS.md](./PANELS.md) — per-panel layout, features, and business flow
- [KEY_FLOWS.md](./KEY_FLOWS.md) — booking lifecycle, payment, onboarding (cross-cutting)
- [FRONTEND_OVERVIEW.md](./FRONTEND_OVERVIEW.md) — frontend module map
- [FEATURE_MATRIX.md](./FEATURE_MATRIX.md) — feature coverage per role

---

## 1. High-level topology

```
 ┌────────────────────────────────────────────────────────────────┐
 │                    Single React codebase                       │
 │   ┌─────────┐      ┌──────────────┐      ┌─────────────────┐   │
 │   │   Web   │      │    Mobile    │      │     Desktop     │   │
 │   │  (PWA)  │      │ (Capacitor)  │      │     (Tauri)     │   │
 │   └────┬────┘      └──────┬───────┘      └────────┬────────┘   │
 │        └──────────────────┼───────────────────────┘            │
 └────────────────────────── │ ───────────────────────────────────┘
                             │  HTTPS  ·  JWT (HS256)
                             ▼
 ╔════════════════════════════════════════════════════════════════╗
 ║                    Cloud Run  ·  np / prod                     ║
 ║                                                                ║
 ║   ┌──────────┐   ┌──────────┐   ┌───────────────┐              ║
 ║   │ identity │   │ customer │   │  marketplace  │ ◄─ booking   ║
 ║   └──────────┘   └──────────┘   └───────▲───────┘    owner     ║
 ║                                         │  PUT status=Paid     ║
 ║   ┌──────────┐   ┌────────────────┐     │                      ║
 ║   │ payment  │──►│  notification  │─────┘                      ║
 ║   └────┬─────┘   └────────▲───────┘                            ║
 ║        │                  │  translate body                    ║
 ║        │                  │                                    ║
 ║   ┌────▼─────┐   ┌────────┴─────┐                              ║
 ║   │ support  │──►│ ai-assistant │  ◄── support summarize       ║
 ║   └──────────┘   └──────────────┘                              ║
 ╚══════════════════════════════ │ ═══════════════════════════════╝
                                 ▼
                ┌──────────────────────────────────────┐
                │   Postgres per service  (× 7)        │
                │   Neon on np  ·  Cloud SQL on prod   │
                └──────────────────────────────────────┘

 External integrations
   payment       →  Stripe · PayPal · Square (+ Cash App) · Braintree/Venmo ·
                    Adyen · Google Pay · Apple Pay · generic REST · manual/offline
                    (admin-configured at runtime; see §7)
   ai-assistant  →  OpenAI  ·  Google Places  ·  Google Geocoding
   notification  →  FCM  ·  APNs  ·  SMTP  ·  SMS
```

Notes:

- All client traffic is direct service-to-service over HTTPS with a shared-secret JWT.
  No API gateway is in front; the frontend's `*Client.ts` modules each target their
  service's base URL.
- Cross-service calls are minimal and one-directional: payment writes the `Paid`
  transition back to marketplace and fires a notification; support and notification
  consume AI; identity is the auth root and is called by all the others only via
  token verification (the shared `SERVICAS_IDENTITY_JWT_SECRET`).
- Each service has its own Postgres database; there is no shared schema.

---

## 2. Frontend layer

Single React 19 + TypeScript + Vite codebase under [src/](../src/).

```
   src/main.tsx
        │
        ▼
   apps/web · WebShell
        │
        ▼
   src/app/App.tsx          (router + chrome)
        │
        ▼
   features/app/usePortalSession   (role → portalRoute)
        │
        ├──►  CustomerWorkspace
        ├──►  ProviderWorkspace
        ├──►  AdminWorkspace
        ├──►  SupportWorkspace
        └──►  RegionalWorkspace
```

- Each panel is a lazy-loaded React component under [src/features/portals/](../src/features/portals/)
  (or `src/features/customer/CustomerWorkspace.tsx` for the customer-facing one).
- All HTTP I/O goes through `features/*/⁠*Client.ts` modules. They attach the JWT from
  secure storage (web localStorage, mobile Capacitor Secure Storage, desktop Tauri store)
  and target one backend each.
- Mobile is **not** a rewrite — Capacitor wraps the same Vite bundle in a native WebView
  (`capacitor.config.ts`). Desktop wraps it with Tauri (`src-tauri/tauri.conf.json`).
  Per memory rule: never propose React Native / Swift / Electron rewrites.
- **Navigation is data-driven and feature-gated.** `features/portals/portalConfig.ts`
  declares every panel's tabs/actions; `features/app/usePortalNavigation.ts` renders them
  and hides any section an admin has disabled (see §7). The customer/provider panels read
  `GET /api/v1/feature-flags/public` to drop disabled sections and capabilities.
- **Payments tokenize client-side.** `features/payments/paymentWidgets/` is a lazy-loaded
  registry (Stripe Elements, PayPal Buttons, Square Web Payments, manual) keyed by provider
  kind; `Checkout.tsx` lists the enabled providers for the market, tokenizes with the chosen
  SDK, then calls payment-service. No card data touches Servicas servers.

---

## 3. Authentication & authorization

- `identity-service` issues JWTs at `/api/v1/auth/login` and `/api/v1/auth/refresh`.
- Token payload includes `sub` (userId), `role` (`CUSTOMER` | `PROVIDER` | `ADMIN` |
  `SUPPORT` | `REGIONAL_MANAGER`), and account state.
- Every other service runs `JwtAuthenticationFilter` against `SERVICAS_IDENTITY_JWT_SECRET`
  and rejects unauthenticated traffic except for explicit `permitAll` matchers
  (e.g. `GET /api/v1/feature-flags/public`).
- Per-request scoping: each service pulls `sub` from the security context and uses it
  to filter `findByCustomerUserId(...)` / `findByProviderUserId(...)` queries. Cross-user
  reads return `404` rather than `403` to avoid leaking existence.
- Role gates inside service code enforce business rules — e.g. only `ADMIN` / `SUPPORT`
  can drive a booking transition for an external provider (`ai:` or `svc:` prefixed
  providerIds). See [KEY_FLOWS.md](./KEY_FLOWS.md#booking-state-machine).

---

## 4. Cross-service contracts

| From               | To                          | Trigger                                       | Endpoint                                          |
|--------------------|-----------------------------|-----------------------------------------------|---------------------------------------------------|
| payment-service    | service-marketplace-service | charge succeeded                              | `PUT /api/v1/marketplace/bookings/{id}/status`    |
| payment-service    | notification-service        | charge failed / succeeded                     | `POST /api/v1/notifications/payments`             |
| notification-service | ai-assistant-service      | translate outbound message                    | `POST /api/v1/ai/translate`                       |
| support-case-service | ai-assistant-service      | summarize ticket / suggest reply              | `POST /api/v1/ai/support/summarize`               |

Service base URLs are configured via env vars: `SERVICAS_MARKETPLACE_BASE_URL`,
`SERVICAS_NOTIFICATION_BASE_URL`, `SERVICAS_AI_BASE_URL`, etc. Local profile points
all of them at `http://localhost:<port>`; np uses Cloud Run service URLs from Google
Secret Manager.

---

## 5. Deployment

```
                          ┌─────────────────────────────┐
   Local dev ─ PR to main─►│  GitHub Actions              │── ► Cloud Run · np
                          │  test + build + deploy (np)  │
                          └─────────────────────────────┘

                          ┌─────────────────────────────┐
   Merge to main ────────►│  test + build (no deploy)    │
                          └─────────────────────────────┘

                          ┌─────────────────────────────┐
   workflow_dispatch ────►│  build + deploy (prod)       │── ► Cloud Run · prod
                          └─────────────────────────────┘
```

- Every backend repo has `.github/workflows/ci-cd.yml`.
- PR to main on a backend = build + test + deploy to **np** (non-prod).
- Merge to main = test + build only. **Prod deploys are manual via `workflow_dispatch`.**
- Secrets live in Google Secret Manager (`servicas-*` prefix) and are mounted into the
  Cloud Run service at boot. The shared JWT secret is `SERVICAS_IDENTITY_JWT_SECRET`.
- Postgres on np uses Neon (cheapest viable per pilot rule); prod uses Cloud SQL.
- Hibernate `ddl-auto: update` keeps schema drift in sync on np — no Flyway / Liquibase.

See [BACKEND_SERVICES.md](./BACKEND_SERVICES.md#deployment-conventions) for
service-by-service env and port mapping.

---

## 6. Tech stack at a glance

| Layer                | Choice                                                     |
|----------------------|------------------------------------------------------------|
| Frontend framework   | React 19 + TypeScript 5.7 + Vite 5.4                       |
| Mobile wrapper       | Capacitor 8 (iOS, Android) — same bundle in WebView        |
| Desktop wrapper      | Tauri 2 (macOS, Windows, Linux) — Rust shell + Vite bundle |
| Unit / component test| Vitest 2 + Testing Library                                 |
| E2E test             | Playwright                                                 |
| Backend framework    | Spring Boot 4.0.x (Java 24 toolchain)                      |
| Persistence          | Spring Data JPA + Hibernate `ddl-auto: update` (+ schema.sql on prod `validate`) |
| Database             | PostgreSQL (Neon on np, Cloud SQL on prod)                 |
| Auth                 | JWT, HS256, shared `SERVICAS_IDENTITY_JWT_SECRET`          |
| Payments             | Admin-configured providers (Stripe / PayPal / Square / Braintree / Adyen / Google Pay / Apple Pay / generic REST / manual) behind one internal contract; secrets AES-256-GCM at rest |
| AI                   | OpenAI (chat, vision, transcription)                       |
| Geo / business search| Google Places + Geocoding                                  |
| Push delivery        | FCM / APNs via PushDeliveryService                         |
| Runtime              | Google Cloud Run                                           |
| CI/CD                | GitHub Actions                                             |
| Secrets              | Google Secret Manager                                      |

---

## 7. Runtime configuration — feature enablement & payment providers

Two admin-driven systems let operators reshape the product at runtime without a redeploy:

**Feature enablement** (identity-service `feature_flags` table, admin tabs *Customer
features* / *Provider features*). Flags are keyed `customer.section.<tab>` /
`provider.section.<tab>` (whole-tab visibility, default-on) and capability flags such as
`customer.media.photo` or `customer.providers.chat` (default-off). The public endpoint
`GET /api/v1/feature-flags/public` exposes the `customer.*` and `provider.*` subsets; the
frontend hides disabled tabs in `usePortalNavigation` and disabled controls in the panels.

**Dynamic payment providers** (payment-service `payment_provider_configs` table, admin tab
*Payment providers*). An admin adds *any* payment type by picking a gateway kind, entering
credentials (encrypted with `SERVICAS_PAYMENT_CONFIG_ENCRYPTION_KEY`), and scoping it to
markets/currencies. `PaymentProviderRegistry` resolves a charge as *enabled DB config →
static env bean → pending stub*; `PaymentProviderFactory` builds the adapter per kind.
Async settlement arrives via signed webhooks at `POST /api/v1/payments/webhooks/{code}`.
See [BACKEND_SERVICES.md#payment-service](./BACKEND_SERVICES.md#payment-service).
