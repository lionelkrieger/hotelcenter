# Copilot instructions (reservation system)

## Source of truth
- **Primary source of truth:** `reservation-system-spec.md`.
- Supporting doc: `AGENTS.md` (repo state + engineering conventions).
- VPS/deploy checklist: `vps-setup.md`.

If any instruction in this file conflicts with `reservation-system-spec.md`, **follow `reservation-system-spec.md`**.

---

## Repo state (important)
- This repository is currently in a **specs-only** phase.
- Do **not** scaffold or implement application code unless the user explicitly asks to start implementation.
- When asked to change requirements or behavior, update `reservation-system-spec.md` (and keep `AGENTS.md` high-level; don't duplicate the whole spec into it).

---

## Product scope guardrails
Keep the system "much simpler" than HotelDruid.

### Goals (must support)
- Multi-property hotel reservation system with:
  - **Internal inventory at room level** (physical rooms).
  - **Guest-facing booking by room type only** (never expose room numbers/room IDs).
  - Room taxonomy (amenities, views, bed configs, accessibility, etc.).
  - White-label booking endpoints on customer-controlled subdomain (CNAME-based setup).
- Deep **Google Hotel Center** integration:
  - Hotel List feed.
  - Landing Pages / Point of Sale.
  - **ARI (Availability, Rates, Inventory)** push model.
  - LoG operational workflows.
- Expose versioned **APIs** + **webhooks** for third-party systems.

### Non-goals (avoid implementing unless explicitly required)
- Full accounting / procurement / advanced CRM / staff scheduling.
- Deep housekeeping beyond an "out of service" flag.
- Complex rule engines (yield management, advanced promos/packages) in v1.

---

## Domain invariants and business rules (do not violate)
- **Stable IDs forever:** `property_id`, `room_type_id`, `rate_plan_id`, `reservation_id` must never be reused once exposed externally (Google, APIs, emails).
- Prefer **UUIDs** for primary keys.
- Prefer **soft deletion** (status/inactive) for entities that must keep stable IDs.
- **No double-booking:** room allocations for the same room must never overlap.
- **Hold TTL:** reservation holds are **fixed at 10 minutes** (MVP).
- **Guest search requires child ages** when child pricing mode requires it (see spec).

---

## Architecture requirements (non-negotiable)
When implementation begins, follow the separation-of-concerns model described in the spec.

### Layers (required)
- **Domain:** pure business rules/invariants only (no HTTP/DB/vendor SDKs).
- **Application:** use cases (create hold, confirm booking, publish ARI, process PayFast ITN).
- **Infrastructure:** DB repos, queue implementation, email/storage, Google/PayFast adapters.
- **Interfaces:** HTTP controllers/routes, webhooks, CLI/admin jobs.

**Rule:** API handlers and workers must call the **same application services**. No duplicated business logic.

---

## Reliability, idempotency, and operations (non-negotiable)
- Use a **durable relational DB** with strong transactions (recommended: Postgres).
- Use **background jobs/queue** for:
  - Google ARI publishing (batching, retries/backoff)
  - webhook processing
  - domain verification + TLS provisioning
  - email delivery
- Enforce **idempotency at the application layer** for:
  - reservation confirmation transitions
  - webhook processing (PayFast ITN)
  - ARI publish jobs
- **Persist diagnostic artifacts** for replay/debug:
  - inbound webhook deliveries (raw payload + metadata)
  - outbound Google ARI payloads + responses

### Logging / correlation
- All logs must be **structured JSON**.
- Every request/job must have a `correlation_id` (or `request_id`) and it must:
  - be returned to clients
  - propagate into background jobs
  - be attached to stored webhook and ARI artifacts

---

## Google Hotel Center requirements (implementation expectations)
- Map IDs consistently:
  - Google `PropertyID` ← our property stable ID
  - Google `RoomID` ← our room type stable ID (guest-facing product)
  - Google `PackageID` ← our rate plan stable ID
- ARI publisher must be event-driven + queued:
  - batch by property/date range
  - throttle
  - retry with exponential backoff + jitter
  - store last success/error per message type
- Keep payload sizes and rate limits in mind (see spec) and implement safe batching.

---

## Guest-facing UX constraints (MVP)
Implement only the UX described in the spec:
- Org booking landing page with branding (logo + banner) and property dropdown.
- Search collects: dates, adults, children, **child ages** (when applicable).
- Results show RoomTypes with exactly **1 image** (MVP), description, amenities, and multiple rate options (RatePlans) with prices.
- Never expose physical room inventory details to guests.

---

## Payments and analytics constraints (MVP)
- PayFast is the first gateway:
  - Redirect flow with server-side ITN verification.
  - Never trust browser redirects as proof of payment.
  - Store webhook deliveries and dedupe retries.
- GTM ecommerce:
  - Fire `purchase` on **booking confirmed**.
  - Prevent double-firing (server-side guard and/or client guard).

---

## How to work in this repo (practical)
- Prefer minimal, spec-driven changes. Don't invent extra features.
- If the user request changes product behavior, update `reservation-system-spec.md` first.
- When coding begins:
  - Fail fast on invalid config (environment-driven validation).
  - Use migrations for all DB schema changes.
  - Add tests only where the repo already uses tests; keep them focused on the change.

---

## When ambiguous
- Ask 1–3 clarifying questions.
- Otherwise default to the simplest interpretation that matches `reservation-system-spec.md` and the "keep it simpler than HotelDruid" constraint.
