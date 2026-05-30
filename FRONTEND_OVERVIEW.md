# Frontend Overview

## Purpose

Servicas Web is a multi-role frontend for a home services marketplace. One app serves five main roles:

- Customer
- Provider
- Admin
- Support
- Regional manager

The app is intentionally feature-rich on the frontend side, with local persistence used to simulate cross-role system behavior.

## App Structure

Core entry points:

- App shell: [src/app/App.tsx](src/app/App.tsx)
- Global styling: [src/app/styles.css](src/app/styles.css)

Feature areas:

- Auth: [src/features/auth](../src/features/auth)
- Availability: [src/features/availability](../src/features/availability)
- AI assistant: [src/features/ai-assistant](../src/features/ai-assistant)
- Booking: [src/features/booking](../src/features/booking)
- Location: [src/features/location](../src/features/location)
- Profile: [src/features/profile](../src/features/profile)
- Portals: [src/features/portals](../src/features/portals)
- Admin persistence: [src/features/admin](../src/features/admin)
- Support tickets: [src/features/support](../src/features/support)
- Payments: [src/features/payments](../src/features/payments)
- Messaging: [src/features/messages](../src/features/messages)
- Reviews: [src/features/reviews](../src/features/reviews)
- Analytics: [src/features/analytics](../src/features/analytics)

## Frontend State Model

The frontend uses React state at the app-shell level and local feature services for persistence.

Persisted browser storage currently covers:

- auth sessions
- registered accounts
- customer profile data
- bookings
- admin market overrides
- audit logs
- support tickets
- receipts
- message threads
- reviews

## Design Direction

The interface uses a dashboard-like application shell with:

- left navigation by portal feature group
- sticky top bar
- panel/grid content layout
- light and dark theme toggle
- toast notifications
- local quick search for portal actions

## What Counts As “Real” In The Frontend

These parts are interactive and persisted:

- booking creation
- provider job actions
- admin override actions
- support ticket creation and status changes
- invoice payment and receipts
- messaging threads
- review submission
- profile persistence

These parts are still mostly presentational:

- provider documents and licensing workflows
- deep market rule editing
- advanced feature-flag controls
- map drawing/routing
- notification delivery outside the browser UI

## Test Coverage

Testing includes:

- unit tests for AI matching
- unit tests for availability logic
- unit tests for portal configuration coverage
- unit tests for booking, payments, and messaging services
- app-level tests for auth, portal switching, booking flow, profile flow, and core interactions

## Recommended Next Frontend Work

- break [App.tsx](src/app/App.tsx) into smaller portal-specific components
- add richer empty/loading/error states around the newer persisted flows
- add more end-to-end tests for payments, admin approvals, and messaging
- move from local storage simulation to a real API layer when backend work begins
