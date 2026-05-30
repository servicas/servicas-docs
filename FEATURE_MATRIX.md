# Feature Matrix

This file tracks which frontend features are implemented as real local workflows versus still mostly presentational.

## Customer

| Feature | Status | Notes |
|---|---|---|
| Login/signup | Implemented | Local auth/session flow |
| Saved homes | Implemented | Persisted profile state |
| Location-based catalog | Implemented | Availability filtered by address and market |
| AI recommendation | Implemented | Keyword/intent-style matching |
| Booking flow | Implemented | Multi-step booking with provider choice |
| Booking history | Implemented | Persisted via booking service |
| Rebook | Implemented | Pre-fills from prior booking |
| Payments | Implemented | Local invoice payment and receipt storage |
| Refund/dispute request | Implemented | Local state only |
| Support tickets | Implemented | Persisted local ticket records |
| Provider chat | Implemented | Shared local message threads |
| Reviews | Implemented | Review submission and provider rating aggregation |
| Favorites | Partial | UI entry point only |
| Tips | Partial | UI only |

## Provider

| Feature | Status | Notes |
|---|---|---|
| Job inbox | Implemented | Drawn from saved customer bookings |
| Accept/decline | Implemented | Updates booking state |
| Quote submission | Implemented | Updates booking quote fields |
| Invoice submission | Implemented | Updates booking invoice fields |
| AI nearby-job recommendations | Implemented | Scores nearby/open jobs by coverage and urgency |
| AI search-demand recommendations | Implemented | Uses saved frontend market/search signals, not live Google API |
| Customer chat | Implemented | Shared message threads |
| Ratings | Implemented | Aggregated from reviews |
| Coverage/pricing/schedule views | Partial | Mostly presentational with some linked data |
| Document workflows | Partial | Display-oriented only |

## Admin

| Feature | Status | Notes |
|---|---|---|
| Market service overrides | Implemented | Persisted by service-region key |
| Audit log | Implemented | Tracks admin actions locally |
| Provider approval queue | Implemented | Approve/suspend actions persisted |
| AI review panels | Partial | Mixed real data and presentation |
| Market rule editing | Partial | Display-oriented only |

## Support

| Feature | Status | Notes |
|---|---|---|
| Ticket queue | Implemented | Shared local ticket records |
| Ticket status transitions | Implemented | Open, Pending, Escalated, Resolved |
| Ticket assignment and priority | Implemented | Owner and priority persist locally |
| SLA targets and internal notes | Implemented | Due dates and support-only notes persist locally |
| Ticket messaging | Implemented | Shared ticket threads |
| Safety and escalation workflow | Implemented | Escalation status and safety flags persist locally |
| Dispute/refund handling | Implemented | Connected to ticket and refund status |

## Regional Manager

| Feature | Status | Notes |
|---|---|---|
| Market demand metrics | Implemented | Derived from bookings/tickets/local market data |
| Supply/completion/cancellation signals | Implemented | Derived analytics |
| Launch planning | Implemented | Saved readiness checklist, decision, and notes by market |
| Supply planning | Implemented | Recruiting targets, active providers, support capacity |
| Global market settings | Implemented | Saved payment/localization/document-readiness fields |

## Shared Platform

| Feature | Status | Notes |
|---|---|---|
| Theme toggle | Implemented | Local preference |
| Quick search | Implemented | Portal action search |
| Toast notifications | Implemented | Frontend-only |
| Cross-role local persistence | Implemented | LocalStorage-backed |
| Backend API integration | Not implemented | Future work |
