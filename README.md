# Servicas — Documentation Index

Servicas is a multi-panel home-services marketplace. The platform is a single React
codebase wrapped for web / mobile (Capacitor) / desktop (Tauri) on top of seven
Spring Boot microservices on Google Cloud Run.

## Business

- **[BUSINESS_OVERVIEW.md](./BUSINESS_OVERVIEW.md)** — investor-facing brief: what Servicas is, target market, revenue model, GTM, competitive landscape, roadmap.

## Architecture

- **[ARCHITECTURE.md](./ARCHITECTURE.md)** — system topology, auth, deployment.
- **[BACKEND_SERVICES.md](./BACKEND_SERVICES.md)** — per-service reference for all 7 backends.
- **[KEY_FLOWS.md](./KEY_FLOWS.md)** — booking lifecycle, payment + retry, provider onboarding, support tickets.

## Panels

- **[PANELS.md](./PANELS.md)** — Customer / Provider / Admin / Support / Regional workspaces with sequence diagrams.
- **[PORTAL_WORKFLOWS.md](./PORTAL_WORKFLOWS.md)** — short-form workflow notes (legacy).
- **[FEATURE_MATRIX.md](./FEATURE_MATRIX.md)** — feature coverage per role.

## Frontend

- **[FRONTEND_OVERVIEW.md](./FRONTEND_OVERVIEW.md)** — module map and feature folders.
- **[INSTALL_MOBILE.md](./INSTALL_MOBILE.md)** — Capacitor (iOS / Android) setup.
- **[INSTALL_DESKTOP.md](./INSTALL_DESKTOP.md)** — Tauri (macOS / Windows / Linux) setup.
- **[platform-migration.md](./platform-migration.md)** — multi-platform notes.

## Quality

- **[customer-smoke-tests.md](./customer-smoke-tests.md)** — QA scenarios.
- **[customer-backend-parity-audit.md](./customer-backend-parity-audit.md)** — UI ↔ backend feature parity for the customer panel.
- **[non-customer-backend-parity-audit.md](./non-customer-backend-parity-audit.md)** — same audit for admin / provider / support panels.

## Repo layout

```
servicas-web/                    ← this repo (React + Vite + Capacitor + Tauri)
└── docs/                        ← you are here

servicas-backends/
├── identity-service/            ← auth, users, roles, feature flags
├── customer-service/            ← addresses, prefs, favorites
├── service-marketplace-service/ ← catalog, providers, bookings
├── support-case-service/        ← tickets, disputes
├── payment-service/             ← invoices, charges, refunds
├── notification-service/        ← push, email, SMS, threads
└── ai-assistant-service/        ← matching, translation, vision
```
