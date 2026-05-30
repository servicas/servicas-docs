# Frontend Overview

## Purpose

Servicas Web is a single React codebase that serves all five roles — **Customer,
Provider, Admin, Support, Regional manager** — and is wrapped for web (PWA), mobile
(Capacitor) and desktop (Tauri) without a rewrite. See
[ARCHITECTURE.md](./ARCHITECTURE.md) for the system view and
[PANELS.md](./PANELS.md) for per-panel detail.

The app is **backend-backed**: every feature talks to one of the seven Spring Boot
services through a typed `*Client.ts` module. There is no mock data layer — panels render
real data or an explicit empty/error state (the one sanctioned exception is the
payment-service pending stub when no gateway is configured).

## App structure

Core entry points:

- App shell / router: [src/app/App.tsx](../src/app/App.tsx)
- Global styling: [src/app/styles.css](../src/app/styles.css)
- Session + role routing: [src/features/app/usePortalSession.ts](../src/features/app/usePortalSession.ts)
- Navigation (feature-gated): [src/features/app/usePortalNavigation.ts](../src/features/app/usePortalNavigation.ts)
- Portal definitions (tabs/actions per role): [src/features/portals/portalConfig.ts](../src/features/portals/portalConfig.ts)

Feature areas under [src/features/](../src/features/):

- Auth: [auth](../src/features/auth) — login/signup/reset, `PortalAuthGateway`, secure token storage
- Customer workspace: [customer](../src/features/customer) — dashboard, providers hub, payments hub, profile
- Booking: [booking](../src/features/booking) — service picker → provider → confirmation
- Portals: [portals](../src/features/portals) — Provider / Admin / Support / Regional workspaces
- Admin: [admin](../src/features/admin) — feature-enablement panels, payment-provider config, market/catalog/AI panels
- Payments: [payments](../src/features/payments) — client, `Checkout`, and the lazy provider-SDK widget registry
- Support: [support](../src/features/support) · Messaging: [messages](../src/features/messages) · Reviews: [reviews](../src/features/reviews)
- AI assistant: [ai-assistant](../src/features/ai-assistant) · Location: [location](../src/features/location) · Analytics: [analytics](../src/features/analytics)

## Data flow

- **Transport** — each backend has a `*Client.ts` (e.g. `paymentClient.ts`,
  `marketplaceClient.ts`) that attaches the JWT and targets that service's base URL,
  resolved at runtime by `config/runtimeConfig.ts` (`*_BASE_URL`). No API gateway.
- **App-shell state** — `App.tsx` + the `features/app/use*` hooks
  (`useProtectedPortalData`, `usePortalActions`, `usePortalNavigation`) own session,
  fetched data, and action handlers; data is loaded from the services and refreshed on a
  timer, not persisted as a simulation.
- **Browser storage** is used only for cross-cutting client state: the auth session/JWT
  (secure storage per platform), theme, and small UI preferences.

## Feature gating

Navigation and controls are driven by admin feature flags
([featureFlagService.ts](../src/features/customer/featureFlagService.ts)). The customer and
provider portals fetch `GET /api/v1/feature-flags/public`; `usePortalNavigation` drops any
disabled section and panels hide disabled capabilities. Admins manage these from the
*Customer features* / *Provider features* tabs. See
[ARCHITECTURE.md §7](./ARCHITECTURE.md#7-runtime-configuration--feature-enablement--payment-providers).

## Payments on the client

`features/payments/paymentWidgets/` is a registry keyed by provider **kind**, each entry a
**lazy-loaded** widget (own Vite chunk): Stripe Elements, PayPal Buttons, Square Web
Payments, and a manual/generic fallback. `Checkout.tsx` fetches the enabled providers for
the market, renders the chosen widget, tokenizes client-side, then calls
`payInvoice` / `sendTip`. Real card data never reaches Servicas servers.

## Design direction

Dashboard-style shell: left navigation by portal feature group, sticky top bar, panel/grid
content, light/dark theme toggle, toast notifications, and a local quick-search over portal
actions. Per the platform rule, panels show **empty / error states only** — never canned
fallback data.

## Test coverage

Vitest + Testing Library and Playwright (e2e):

- unit tests: AI matching, availability logic, portal-config coverage, booking / payments /
  messaging services, the payment widget registry, and feature-flag gating
- app-level tests: auth, portal switching, booking flow, profile flow, payments, and core
  interactions

## Recommended next frontend work

- continue breaking [App.tsx](../src/app/App.tsx) into smaller portal-specific components
- add dedicated client widgets for the remaining provider kinds (Braintree, Adyen, native
  Google/Apple Pay wallets) — they currently fall back to the manual widget
- broaden e2e coverage for payments (sandbox), admin approvals, and messaging
