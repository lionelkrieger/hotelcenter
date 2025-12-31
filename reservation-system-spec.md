# Reservation System (Google Hotels + APIs)

Date: 2025-12-30

## Goals
- Multi-property hotel reservation system with:
  - Internal inventory managed at **room level** (true physical rooms).
  - Guest-facing booking by **room type only** (no exposure of specific room numbers/inventory details).
  - “Room taxonomy” to describe rooms/room types (amenities, views, bed configs, accessibility, etc.).
  - White-label guest booking endpoints on a customer-controlled subdomain (CNAME-based setup).
- Deep integration with **Google Hotel Center**:
  - Hotel List feed (static property metadata).
  - Landing pages / Point of Sale (POS) definitions.
  - **ARI (Availability, Rates, Inventory)** push model for near-live updates.
  - Support Live on Google (LoG) operational workflows.
- Expose **APIs/endpoints** for third-party hotel management systems (PMS/CRS/channel manager) to:
  - Import/export reservations.
  - Push inventory/rates/restrictions.
  - Receive change notifications (webhooks).

## Non-goals (keep system “much simpler” than HotelDruid)
- Full accounting, procurement, advanced CRM, staff scheduling.
- Deep housekeeping/maintenance modules beyond “out of service” flags.
- Complex rule engines in v1 (yield management, advanced promos, packages) unless required by Google ARI setup.

---

## Tech stack sanity check (Dec 2025)

This product’s “hard requirements” (Google ARI push + retries, inbound payment webhooks, multi-tenant APIs, and custom domains with automated TLS) strongly favor a **cloud SaaS backend + background workers + a durable database**, rather than a WordPress/PHP-first architecture.

### Non-negotiables (regardless of language/framework)
- **Relational DB** with strong transactions (recommended: Postgres).
- **Background jobs/queue** for:
  - Google ARI publisher (batching, retry/backoff)
  - email delivery
  - domain verification + TLS provisioning workflows
- **Idempotent webhook processing** (PayFast ITN now; more gateways later) with replay-safe state transitions.
- **Object storage** for uploads (org logo/banner, room type image).
- **Observability**: structured logs + metrics + per-property integration error visibility.
- **Secret management** for gateway credentials and API keys.

### Recommended option (fastest path, best ecosystem fit)
- **TypeScript** across backend + frontend.
- Backend: a Node.js API framework with strong typing and middleware ecosystem.
- Frontend: a React-based app for the booking UI (server rendering supported for SEO/perf).
- Queue: Redis-backed queue or managed queue service (either is fine; must support delayed retries).
- DB: Postgres.

Why this remains the best fit:
- Excellent support for webhooks, SDK integrations, and JSON APIs.
- Shared types between pricing/availability quoting and admin tools.
- Straightforward worker model for ARI publishing.

### Alternative option (also good)
- **Python** backend (FastAPI-style) + background workers (Celery/RQ-style) + Postgres.
- Frontend remains a modern JS UI.

Choose this if your team is significantly stronger in Python; it still meets all constraints as long as queues/observability/idempotency are first-class.

### Explicit non-recommendation
- Using WordPress/WooCommerce/PHP as the core engine is not recommended for this system because ARI publishing and webhook reliability need durable queues, idempotent processing, and operational tooling. WordPress can still be used as a marketing site that links/embeds the booking UI.

---

## Engineering requirements: separation of concerns + debuggability

Goal: keep the codebase easy to maintain as features (Google ARI, payments, loyalty, custom domains) grow, and make production issues diagnosable from logs + stored artifacts.

### Separation of concerns (required)
- The codebase must be organized into clear layers/modules:
  - **Domain**: core business objects + invariants (reservations, holds, availability, pricing, loyalty). No HTTP, no DB, no vendor SDKs.
  - **Application**: use-cases/services (create hold, confirm booking, publish ARI, process ITN) orchestrating domain logic.
  - **Infrastructure**: DB repositories, queue implementation, email provider, object storage, Google/PayFast adapters.
  - **Interfaces**: HTTP controllers/handlers, webhooks, CLI/admin jobs.
- External integrations (Google ARI, PayFast ITN) must be isolated behind adapters with stable internal interfaces.
- Business rules must not be duplicated between API handlers and workers; both call the same application layer.

### Debuggability & operations (required)
- **Structured logging** (JSON) for API + workers.
- Every inbound request/job must have a `request_id`/`correlation_id` and it must be:
  - returned to clients (for support)
  - propagated into background jobs spawned from that request
- **Error handling** must be consistent:
  - safe, non-leaky error messages to clients
  - full diagnostic context in server logs (including stack traces in non-prod)
- **Auditability**:
  - record who did what for staff/admin changes (rates, inventory, cancellations, domain settings)
  - store inbound webhook deliveries and outbound Google ARI payloads + responses for replay/debug
- **Idempotency** enforced at application layer for:
  - booking confirmation transitions
  - webhook processing
  - ARI publish jobs

### Maintainability guardrails (required)
- Configuration must be environment-driven and validated at startup (fail fast on missing/invalid settings).
- DB schema changes must be managed by migrations.
- Provide a minimal “runbook” section in docs covering:
  - how to trace a booking end-to-end via `correlation_id`
  - how to re-drive ARI publishing for a property/date range
  - how to inspect webhook history for a reservation/payment

---

## Canonical domain model (internal)

### Multi-tenancy
- **Account** (customer organization) owns multiple **Properties**.
- Every data row is scoped by `account_id` and `property_id` as appropriate.

### Users, roles, and invitations (staff/admin)

#### Roles (minimal RBAC)
- `master_admin` (platform operator): can create organizations, assist support, view system health.
- `org_admin` (customer admin): can manage properties, staff, billing settings, branding, domains.
- `staff` (customer staff): can manage reservations and (optionally) inventory/rates depending on permissions.

#### Org-admin configurable permissions
In addition to the coarse `role`, org admins can grant fine-grained permissions to staff.

- Model permissions as named capabilities (examples):
  - `reservations.read`, `reservations.write`, `reservations.cancel`
  - `guests.read`, `guests.write`
  - `rates.read`, `rates.write`
  - `inventory.read`, `inventory.write`
  - `room_types.write`, `rooms.write`
  - `branding.write`, `domains.write`
  - `integrations.write`, `api_keys.write`

- Recommended UX: “Permission groups” (preset bundles) + optional overrides.
  - Example groups: Front Desk, Manager, Revenue, Housekeeping (later), Admin.

**Rule:** `org_admin` always has all permissions for their account.

#### Suggested tables
- `user`:
  - `user_id`, email (unique), name
  - auth: magic-link enabled
- `account_user`:
  - `account_id`, `user_id`, `role`
  - optional per-property scope if needed later
- `permission`:
  - `permission_key` (string, stable)
  - description
- `permission_group`:
  - `group_id`, `account_id`, name
- `permission_group_permission`:
  - `group_id`, `permission_key`
- `account_user_permission_group`:
  - `account_id`, `user_id`, `group_id`
- `account_user_permission_override` (optional, if you want per-user tweaks)
  - `account_id`, `user_id`, `permission_key`, `effect` (allow|deny)
- `invite`:
  - `invite_id`, `account_id`, `email`, `role`
  - `token_hash`, `expires_at`, `accepted_at`
  - `created_by_user_id`

### Property
- `property_id` (stable, never reused)
- name, address, geo, timezone, currency
- check-in/out times
- Google linkage fields:
  - `google_hotel_list_id` (usually equals internal `property_id`, must be unique “for all time”)
  - map match status, LoG flags

### RoomType (guest-facing)
- `room_type_id` (stable per property)
- property_id
- name, description, capacity rules
- photos/media
- active/inactive
- taxonomy facets (see Taxonomy)

### Room (internal physical inventory)
- `room_id` (stable)
- property_id
- label/number
- `room_type_id` (current sellable type)
- status: active | out_of_service
- taxonomy facets (see Taxonomy)

### RatePlan (Google “PackageID”)
- `rate_plan_id` (stable per property)
- refundable rules, meal plan flags, included values
- active/inactive

Pricing (MVP): a rate plan can either have its own explicit price table, or be derived from a base price via a modifier.
- `pricing_mode`: `explicit_table` | `derived_from_base`
- If `derived_from_base`:
  - `base_rate_plan_id` (usually the property’s BAR / Standard)
  - `modifier_type`: `percent` | `amount_per_night` | `amount_per_stay`
  - `modifier_value`
  - `rounding_rule` (optional, e.g. nearest 1.00)
- Loyalty interaction:
  - `allow_loyalty_discount` (boolean, default true)

Association:
- A property may define many rate plans.
- Each room type explicitly declares which rate plans are offered to guests.

Suggested join table:
- `room_type_rate_plan`:
  - `room_type_id`, `rate_plan_id`, `sort_order`, `active`

### Nightly Rate / Restrictions (per property, room type, rate plan, date)
- rates:
  - base amount per night (and optionally occupancy tiers)
- restrictions:
  - min/max LOS, CTA/CTD, closed/open

Note: even if a rate plan is `derived_from_base`, the system must be able to quote and publish a concrete nightly price per (property, room_type, rate_plan, date, occupancy) for:
- guest quoting consistency
- reservation price snapshots
- Google ARI publishing

### Reservation (guest booking)
- `reservation_id`
- property_id
- guest contact fields
- stay dates: `checkin_date`, `checkout_date` (dates; checkout non-inclusive)
- status: hold | confirmed | cancelled | checked_in | checked_out

### ReservationLine (what the guest buys)
- reservation_id
- room_type_id
- rate_plan_id
- quantity
- pricing snapshot (total, taxes/fees if used)

### RoomAllocation (internal assignment)
- reservation_line_id
- room_id

**Invariant:** no overlapping allocations for the same room on overlapping date ranges.

---

## Room taxonomy (simple but powerful)
Use “attributes/tags” with categories.

- `taxonomy_term`:
  - `term_id`, `property_id`
  - `kind` (amenity|view|bed|accessibility|feature|…)
  - `name`
- `room_taxonomy` (room_id, term_id)
- `room_type_taxonomy` (room_type_id, term_id)

Guest-facing UI shows **room_type_taxonomy** (or derived rollups) but never exposes which rooms have which terms.

---

## Guest-facing UX (minimal but high-quality)

### Flow
1. Arrive on org booking domain (custom subdomain) landing page.
2. Select property (dropdown) if org has multiple properties.
3. Enter search: dates + occupants **including child ages**.
4. See **RoomTypes** list with:
   - 1 image per room type (MVP)
   - description
   - amenities list (room-type level)
   - availability indicator
   - one or more **rate options** (rate plans) with prices
5. Choose a rate option and proceed to guest details + payment (if required).
6. Confirmation page + email.

### Org public landing page (guest)
Requirement: every org has a public booking landing page on their configured custom subdomain.

Must include:
  - logo upload (org-level)
  - banner image upload (org-level)
  - searchable dropdown list of properties in the org
  - if only one property, auto-select and hide dropdown
  - check-in and check-out dates
  - adults count
  - children count + **child ages input** (collected before search)
  - “Search” / “Check availability” leading to room type results

### Branding (org public landing page)
Org admins must be able to configure the org’s public booking landing page:
- Upload **logo** (org-level asset)
- Upload **banner image** (org-level asset)
- (Optional MVP text fields) org display name and short tagline

### Properties (used for guest dropdown)
Org admins must be able to manage the list of properties that appear in the guest property selector:
- Create/edit property: name, address/geo basics, timezone, default currency
- Enable/disable property for public booking
- Set a property slug for URLs (stable once published)

### Room types (content + merchandising)
Org admins (or staff with permission) must be able to manage room types per property:
- Name
- Description
- Amenities (select from amenity taxonomy; stored as room-type attributes)
- Upload **exactly 1 image per room type** (MVP)

### Rate options (rate plans) that modify price
Org admins must be able to create multiple bookable “options” per room type by defining rate plans:
- Rate plan name (e.g., “Pensioners Bed and Breakfast”, “Summer Sale incl Breakfast”)
- What’s included (free-text + optional boolean flags like `includes_breakfast`)
- Eligibility:
  - public
  - member-tier gated
  - promo-code gated
- Pricing rules:
  - nightly prices (or derived from base rate + adjustment) per date range
  - child pricing behavior follows the property’s configured child-pricing mode

Guest results must only show rate plans that are:
- allowed for the room type
- available for the selected dates
- eligible for the guest (or shown as locked with login/signup CTA)

### Availability shown to guest
- For each room type:
  - `available_count = min(available_rooms_each_night_in_range)`
- Guest never sees specific room IDs.

### Room type presentation (guest)
- Each room type shows:
  - name
  - single primary image (MVP)
  - description text
  - amenities list (simple taxonomy terms, e.g., Wi‑Fi, Air conditioning, TV)

### Rate options per room type (guest)
Requirement: room types can show multiple bookable “options” (for example “Pensioners Bed and Breakfast”, “Summer Sale incl Breakfast”), and selecting an option changes the price.

Implementation mapping:
- Each “option” is a `RatePlan` (`PackageID` in Google terms) that is allowed for the room type.
- Prices are stored and returned per (property, room_type, rate_plan, date, occupancy).

UI requirements:
- Under each room type, list the available rate plans for the selected dates/occupancy, each showing:
  - rate plan name
  - price (per stay and/or per night; be consistent)
  - a primary CTA: “Book now”
- If a rate plan is `private` (member tier or promo code):
  - show member price when eligible
  - otherwise show a clear CTA like “Sign up” / “Log in” to unlock

---

## Booking lifecycle + inventory holds

### Reservation states (MVP)
- `hold`: temporary reservation while guest completes checkout (used for redirect payments).
- `confirmed`: booking is final and inventory is committed.
- `cancelled`: booking cancelled by guest/staff/system.
- `expired`: hold timed out (system action).

### Hold duration (MVP)
- Hold TTL is **fixed at 10 minutes**.
- Rationale: long enough to complete payment, short enough to avoid blocking scarce inventory.

### Recommended flow
- `pay_on_arrival`:
  - confirm immediately in a single transaction (no hold required unless you have a multi-step checkout).
- `prepay_full` (redirect gateway like PayFast):
  1) create reservation in `hold` + reserve inventory
  2) redirect to gateway
  3) confirm only when payment succeeds (ITN/webhook)

### Expiry behavior
- Background job expires holds after 10 minutes and releases inventory.
- If payment confirmation arrives after expiry, attempt re-check availability:
  - if available: confirm
  - else: mark as “paid but unconfirmed” for manual resolution (refund workflow can be added later)

---

## Custom subdomain setup (CNAME) + org onboarding

### Requirement
- A **master admin** can create an **Organization/Account** and connect that org to a booking subdomain on the org’s own domain (for example `reservations.hotelname.com`) through an easy guided setup.

### Recommended approach (CNAME to SaaS)
- Customer creates a DNS record:
  - `reservations.hotelname.com CNAME org-slug.book.yoursaas.com`
  - Alternative: `CNAME book.yoursaas.com` plus an onboarding token (see Verification).
- SaaS provisions TLS and serves the booking UI on the custom hostname.

### Setup flow (user-friendly)
1. Master admin creates Organization (Account) in SaaS and enters org admin email.
2. SaaS sends org admin a **magic link** to complete account setup.
3. SaaS generates:
   - suggested subdomain (default: `reservations`) and required DNS record value
   - a verification token (only needed if using a shared CNAME target)
4. Org admin adds DNS record(s):
   - required: `CNAME` for the booking hostname
   - optional verification (pick one):
     - `TXT _yoursaas-verification.reservations.hotelname.com = <token>`
     - or `TXT reservations.hotelname.com = <token>` (if allowed)
5. Org admin clicks “Verify”. SaaS checks:
   - DNS resolves to expected CNAME target
   - if verification enabled, TXT contains expected token
6. SaaS enables the domain + issues certificate.
7. Org admin configures branding and publishes Google landing pages to use the custom hostname.

### Tenant routing
- Every request is routed by `Host` header → `custom_domain` mapping → `account_id`.
- Custom domains are unique across all accounts.

### TLS/Certificates
- Use an automated certificate manager (ACME/Let’s Encrypt) to provision certificates per custom domain.
- If fronted by a CDN/proxy (recommended), prefer provider-managed certificates + origin authentication.

### Security constraints
- Do not allow wildcard custom domains by default.
- Only activate a hostname after DNS verification succeeds.
- Prevent hostile takeover by requiring a TXT token when multiple orgs could point to the same CNAME target.

### Operational constraints
- DNS propagation can take minutes to hours; UI should show clear status and re-check.
- Store domain lifecycle status: `pending_dns` → `verified` → `active` (and `failed`/`disabled`).

---

## Staff onboarding (magic links)

### Requirement
- Master admin invites an org admin via email magic link.
- Org admin can invite staff; staff create profiles via magic links.
- This is the primary mechanism to let the org set up inventory (properties/rooms/room types), rates, and branding.

### Magic link behavior (security + UX)
- Single-use, short-lived tokens (example: 15–60 minutes).
- Store only `token_hash` server-side.
- Rate-limit magic link requests per email/IP.
- If a user already exists, magic link logs them in; if not, it completes signup.

### Minimum onboarding checklist for org admin
- Add properties (or import).
- Define room types + photos.
- Add rooms (internal inventory) and assign to room types.
- Set rate plans (public + private member tiers + promo-code plans).
- Configure branding (logo, primary color token, typography choices if permitted by design system).
- Configure custom booking subdomain + verify.
- Generate/submit Google Hotel List feed + ARI product definitions.

---

## Google Hotel Center integration

### 1) Hotel List Feed (static property metadata)
From your notes:
- XML `<listings>` feed with one `<listing>` per property.
- **Separate file per language** (`<language>` element).
- `<id>` must be **unique for all time**; never reused.
- Hosted on HTTP/HTTPS; can be zipped; BASIC/DIGEST auth supported.
- Google fetch schedule typically weekly.

**System requirements**
- Generate hotel list feeds from canonical Property data.
- Maintain stable `google_hotel_list_id` (do not recycle IDs even if property deleted).
- Feed publishing should be atomic (build in staging directory, then swap/symlink).

### 2) Landing Pages / Point of Sale (POS)
From your notes:
- Define one or more `<PointOfSale id="...">` entries with `<Match …>` criteria.
- Google picks the most specific/best match.
- Recommendation: minimize match criteria (over-restricting reduces Free Booking Links participation).
- `<URL>` supports variables like:
  - `(PARTNER-HOTEL-ID)`, `(CHECKINDAY)`, `(CHECKINMONTH)`, `(CHECKINYEAR)`, `(LENGTH)`
  - `(NUM-ADULTS)`, `(NUM-CHILDREN)` or child-age loop blocks
  - `(USER-LANGUAGE)`, `(USER-CURRENCY)`, `(GOOGLE-SITE)`, `(VERIFICATION)`
- Conditional URL directives supported: `IF-AD-CLICK`, `IF-VERIFICATION`, etc.

**System requirements**
- Provide landing endpoints that accept Google query params and pre-fill the booking UI.
- Handle verification traffic: `(VERIFICATION)=true` should not distort analytics or inventory.
- Decide a canonical landing URL format:
  - Example: `https://book.example.com/h/(PARTNER-HOTEL-ID)?checkin=(CHECKINYEAR)-(CHECKINMONTH)-(CHECKINDAY)&nights=(LENGTH)&adults=(NUM-ADULTS)&children=(NUM-CHILDREN)&lang=(USER-LANGUAGE)&currency=(USER-CURRENCY)&src=(GOOGLE-SITE)&verify=(VERIFICATION)`

### 3) ARI Push (Availability, Rates, Inventory)
From your notes:
- Must send: **Transaction (Property Data)** + **Rate** + **Inventory** + **Availability** before prices show.
- Push-on-change model; Google merges updates; send only deltas.
- POST XML to endpoints under `https://www.google.com/travel/hotels/uploads/...`:
  - property data: `/property_data`
  - rates: `/ota/hotel_rate_amount_notif`
  - inventory: `/ota/hotel_inv_count_notif`
  - availability: `/ota/hotel_avail_notif`
  - plus optional: taxes, promotions, rate_modifications, extra_guest_charges
- Limits in notes:
  - keep `OTA_HotelRateAmountNotifRQ` around 5MB
  - account max 400 msg/sec
- Google returns HTTP 200 for transport success; XML body indicates warnings/errors.

**Key mapping**
- Google `PropertyID` ← our `property_id` (or dedicated external id mapping)
- Google `RoomID` ← our `room_type_id` (guest-facing product)
- Google `PackageID` ← our `rate_plan_id`

**System requirements**
- Implement an outbound “Google ARI publisher” that:
  - listens to internal change events (rates/inventory/room type/rate plan changes)
  - batches by property/date range
  - applies throttling + retry
  - records last successful push state + error diagnostics

**Recommended architecture**
- Use an outbox/event table on commits (reservation created/modified/cancelled; rate changes; room out-of-service).
- A background worker builds ARI XML and POSTs to Google.
- Idempotency: dedupe events by `(property_id, room_type_id, rate_plan_id, date_range, kind)`.

### 4) Live on Google (LoG)
From your notes:
- LoG defaults true for new hotels; can be controlled via Hotel Center UI or Travel Partner API (`hotelViews.list`, `hotelViews.summary`).

**System requirements**
- Track desired LoG state per property.
- Provide an admin toggle and (later) an integration job that reconciles LoG via Travel Partner API.

---

## Third-party integrations (PMS/CRS/channel manager)

### Design principle
- Treat this system as the **source of truth for availability** (or explicitly support “external master” mode per property).
- Provide clean, versioned APIs and webhooks.

### Public REST API (suggested)
Base: `/api/v1`

**Read models**
- `GET /properties`
- `GET /properties/{propertyId}/room-types`
- `GET /properties/{propertyId}/rate-plans`
- `GET /properties/{propertyId}/availability?start=YYYY-MM-DD&end=...&roomTypeId=...`

**Write models**
- `POST /reservations` (create)
- `PATCH /reservations/{id}` (modify guest/dates/status)
- `POST /reservations/{id}/cancel`
- `POST /rates/bulk` (push rates)
- `POST /inventory/bulk` (push inventory adjustments / out-of-service)
- `POST /restrictions/bulk`

### Webhooks
- `reservation.created`
- `reservation.modified`
- `reservation.cancelled`
- `inventory.changed`
- `rate.changed`

### Authentication
- API keys per integration + scoped permissions per property.

---

## Analytics: Google Tag Manager + ecommerce dataLayer

### Goal
- Let org admins add their GTM container and receive clean ecommerce events for bookings.
- Ensure events include the fields you listed: `transaction_id`, `id`, `start_date`, `end_date`.

### Admin configuration (per account or per property)
- `gtm_container_id` (example: `GTM-XXXXXXX`)
- optional: `ga4_measurement_id` if you want to support gtag.js directly (but GTM-first is fine).

### Pricing + policy parameters in analytics
- Include `start_date`/`end_date` and consider adding these custom params for reporting:
  - `property_id`, `property_name`
  - `payment_policy` (`pay_on_arrival`|`prepay_full`)
  - `rate_plan_id`

### Where to fire events
- Fire ecommerce events when the booking is **confirmed**.
  - If the property is configured for **pay on arrival**, confirmation happens at booking time → fire `purchase` immediately.
  - If the property requires **prepayment**, confirmation happens after payment success (for PayFast: ITN `complete`) → fire `purchase` then.
- Use server-rendered confirmation page or a client-side hook that reads the final reservation state from the API.
- Prevent duplicates via a `purchase_event_id` stored server-side and/or a browser-side guard (localStorage).

### Suggested GA4-compatible dataLayer payload (purchase)
Push on booking confirmation:

- `event`: `purchase`
- `ecommerce`:
  - `transaction_id`: reservation confirmation number or payment intent ID (stable)
  - `value`: total amount charged (ZAR)
  - `currency`: `ZAR`
  - `items`: list of booked room types

Include your required extra fields at the top-level (or as custom params):
- `id`: same as `transaction_id` (for systems expecting `id`)
- `start_date`: check-in date (`YYYY-MM-DD`)
- `end_date`: check-out date (`YYYY-MM-DD`)

Item mapping (per room type booked):
- `item_id`: `room_type_id`
- `item_name`: room type display name
- `item_category`: property name or a fixed category like `Hotel`
- `quantity`: number of rooms
- `price`: per-night or per-stay item price (be consistent)

### Optional events
- `begin_checkout` when the guest proceeds from room selection to details.
- `add_payment_info` when redirecting to PayFast.
- `refund` if a paid booking is refunded.

---

## Payments: PayFast (South Africa) integration

### Model
- Create a `payment_intent` for a reservation:
  - `payment_intent_id` (internal)
  - `reservation_id`
  - `amount`, `currency` (ZAR)
  - status: `created` | `redirected` | `paid` | `failed` | `cancelled`

### Org configuration (credentials + test mode)
- Org admin configures PayFast settings (optionally per property):
  - `merchant_id`
  - `merchant_key`
  - `passphrase` (optional)
  - `mode`: `sandbox` | `production`

Notes:
- Store credentials encrypted; never send secrets to the browser.
- `mode` must be selectable per org so customers can test safely before going live.
- Optional override: allow `mode`/credentials per property for groups with separate merchant accounts.

### Redirect flow (hosted PayFast)
1. Guest confirms booking and selects PayFast.
2. Backend creates `payment_intent` and returns a PayFast redirect/form payload.
3. Frontend posts to PayFast with:
  - merchant fields (`merchant_id`, `merchant_key`)
  - `return_url`, `cancel_url`
  - `notify_url` (server-to-server ITN callback)
  - `m_payment_id` (set to `payment_intent_id` or reservation number)
  - `amount`, `item_name`, `item_description`
  - `signature` computed from the parameter string + passphrase.

**PayFast endpoint selection:**
- If `mode=sandbox`, use PayFast sandbox process/validation endpoints.
- If `mode=production`, use PayFast production endpoints.

### ITN (Instant Transaction Notification) callback
- Implement `POST /api/v1/payments/payfast/itn` as the `notify_url`.
- On ITN receipt:
  1. Verify signature.
  2. (Production) verify source IP.
  3. Validate payload with PayFast validation endpoint (server-to-server) if required.
  4. Verify amount matches expected.
  5. Transition `payment_intent` + reservation status.

### Test-mode transaction handling
- Mark each `payment_intent` with `mode` at creation time so it can’t be confused later.
- Keep test payments out of revenue reporting by default.
- Analytics:
  - Do not fire `purchase` for sandbox payments unless the org explicitly enables “track test purchases”.

---

## Payments: extensible gateway architecture (uploadable extensions)

### Requirement
- System must support more gateways than PayFast.
- We want a way to “add a payment extension” (upload a zip) that:
  - declares the credential fields/settings UI
  - implements gateway actions (create checkout, handle webhooks/ITN, refunds if supported)
  - can be enabled per org/property

**Governance:** only `master_admin` can upload/modify/delete payment extensions. Org admins can only enable/disable approved gateways.

### Security reality check (important)
Allowing org admins to upload arbitrary executable code into a SaaS is high-risk.

To keep the product viable:
- Either:
  1) extensions are **vendor-reviewed / marketplace approved** before running, OR
  2) extensions are **non-executable** (declarative) and the platform provides the runtime, OR
  3) extensions run **outside** the SaaS as “connectors” calling our APIs.

For MVP, recommend (2) + (3): manifest-driven UI + external connector option.

### Preferred design: manifest + sandboxed runtime (platform-controlled)

#### Extension package (zip)
- `manifest.json`
  - `id`, `name`, `version`, `author`
  - `capabilities`: `redirect_checkout`, `webhook`, `refund` (subset)
  - `supported_currencies`, `supported_countries`
  - `webhook_path` (relative) and `return_paths`
- `config.schema.json` (JSON Schema)
  - drives the dashboard UI for credential entry (merchant id/key, API token, mode, etc.)
  - includes field types, required flags, help text
- `handler.(js|ts|wasm)`
  - implements a small fixed interface (below)

#### Extension interface (example)
- `getConfigSchema(): JsonSchema`
- `createCheckout(input): { type: 'redirect'|'form_post', url, method, fields }`
- `handleWebhook(request): { status, externalPaymentId, outcome, raw }`
- `refund(input): { status, externalRefundId }` (optional)

#### Execution constraints
- Run extension code in an isolated sandbox (no filesystem, no OS access).
- Strict allowlist outbound network to the gateway’s domains.
- Timeouts, memory caps, request size limits.
- No direct database access: extension gets only the input you pass it; it returns a structured result.

#### Platform responsibilities
- Persist org-level config values for an enabled gateway (encrypted at rest).
- Render config UI from `config.schema.json`.
- Route webhooks to the correct org/gateway:
  - `POST /api/v1/payments/{gatewayId}/webhook` (platform endpoint)
  - platform invokes the extension handler.
- Maintain versioning and rollback:
  - pin orgs to an extension version
  - allow disable/rollback if issues occur.

### Safer alternative: “External connector” mode (recommended for custom gateways)
Instead of uploading code, the hotel group (or you) runs a small connector service that:
- talks to the payment provider
- calls your platform APIs to create/confirm payments
- receives provider webhooks and forwards normalized results to your platform

This avoids executing untrusted code inside the SaaS while still enabling long-tail gateway support.

### MVP scope recommendation
- Implement a fixed set of first-party gateways (PayFast first).
- Add the extension framework with a **master-admin-managed gateway catalog**:
  - master admin uploads and publishes extensions
  - org admins can see the published gateways (for example PayFast, Stitch) and choose what to enable
- Later, add a review/approval pipeline if you want a true marketplace.

### Org admin experience (gateway catalog)
- “Payments” page shows:
  - list of available gateways published by the platform (name, supported currencies, supported countries)
  - status per gateway: disabled | enabled (and enabled scope: org-wide or per property)
  - configuration form generated from `config.schema.json`
- Enabling a gateway requires:
  - org admin entering credentials/settings
  - selecting `mode`: sandbox | production
  - selecting whether prepayment is required (per property setting)

### When to mark the reservation confirmed
- Support a per-property `payment_policy`:
  - `pay_on_arrival`: no online payment; reservation is confirmed immediately.
  - `prepay_full`: online payment required; reservation is confirmed only after payment success.
- PayFast applies to `prepay_full` flows.

### When to emit analytics
- Fire `purchase` when the reservation is in its final **confirmed** state.
  - `pay_on_arrival`: confirmed without payment.
  - `prepay_full`: confirmed after payment.
- If the user returns from PayFast before ITN is processed, show “Processing payment…” and poll until the server confirms.

---

## Pricing configuration (org admin dashboard)

### Pricing display rules (tax/fee inclusion)
Goal: keep guest totals consistent with Google price checks.

Per property, org admin configures:
- `pricing_display_mode`:
  - `total_including_mandatory_taxes_fees` (recommended default)
  - `total_excluding_taxes_fees` (only if the property truly cannot show them upfront)
- `mandatory_taxes_fees` definition (MVP supports a small set):
  - percentage tax on room subtotal (e.g. VAT)
  - fixed fee per night (e.g. tourism levy)
  - fixed fee per stay

Rules:
- If a tax/fee is mandatory and known at booking time, include it in the displayed total when `pricing_display_mode=total_including_mandatory_taxes_fees`.
- Do not invent unknown fees. If fees vary by guest behavior, exclude them and disclose clearly.

### Base price + modifiers (rate options + loyalty) (MVP)
Terminology note: pricing is attached to **RoomType** (the sellable unit). Physical **Room** rows do not carry prices in MVP.

Admin flow (MVP-friendly):
1) Staff creates a room type and sets a **base price** (the “Standard / BAR” rate plan).
2) Staff adds **rate options** by attaching additional rate plans to that room type.
3) Each additional rate plan can either:
   - have its own explicit nightly prices, OR
   - be **derived** from the base price via a modifier (percent or amount).

Examples:
- “Pensioners Bed and Breakfast”: derived from base with `-10%` plus `+amount_per_night` if breakfast is priced separately.
- “Summer Sale incl Breakfast”: derived from base with `-15%` and meal flag set.

Loyalty tier adjustment (dynamic):
- Loyalty tiers define a discount (e.g., Bronze -5%, Silver -10%).
- Apply loyalty discount only when the guest is authenticated/eligible.
- Default stacking rule (MVP):
  - Selected rate plan modifier applies first.
  - Then loyalty discount applies if `allow_loyalty_discount=true` for that rate plan.
  - Promo codes (if used) apply last (or disable loyalty if you prefer); choose one rule and keep it consistent everywhere (UI, emails, Google price checks).

### Child pricing inputs (ages vs count)
Per property, org admin selects `child_pricing_mode`:
- `none`: children priced the same as adults or not supported
- `count_only`: price depends only on number of children (no ages collected)
- `age_bands`: price depends on child ages (collect ages)

If `age_bands`:
- Org admin defines age bands (MVP: up to 3 bands, non-overlapping):
  - example: 0–2, 3–12, 13–17
- Guest UI collects child ages and maps them into bands.

Pricing storage (MVP-friendly):
- Nightly rates can be stored as a matrix keyed by (adults_count, child_band_counts).
- No “extra guest charges” engine is required; this is just a precomputed rate table.

---

## Guest-facing policies (cancellation / deposit / no-show)

### Why
- Policies materially affect guest trust and conversion.
- Google surfaces policies and will quality-check landing page consistency.

### Model (MVP)
Attach policies to `rate_plan` (since different plans often have different rules).

- `cancellation_policy`:
  - `refundable` boolean
  - `free_cancel_until_days` (integer, e.g. 2)
  - `free_cancel_until_time_local` (HH:MM, property timezone)
  - `cancellation_penalty_type`: `none` | `first_night` | `percent` | `flat_amount`
  - `cancellation_penalty_value` (percent or amount)

- `deposit_policy` (MVP)
  - For `prepay_full`, deposit is inherently “100% online payment”.
  - For `pay_on_arrival`, default is **no deposit** in MVP.
  - (Optional later) support deposit amounts/percent for pay-on-arrival.

- `no_show_policy` (display-first MVP)
  - same structure as cancellation penalty

UI requirements:
- Show policy summary on room selection, checkout, and confirmation email.

---

## Webhook security + idempotency (PayFast ITN + future gateways)

### Security requirements
- Always verify authenticity:
  - signature/HMAC (provider-specific)
  - timestamp/replay window (where supported)
  - allowlist source IPs (where supported; PayFast production)
- Never trust browser redirects as proof of payment.

### Idempotency requirements
- Store each inbound webhook/ITN delivery with:
  - `gateway_id`, `external_event_id` (or derived id), `received_at`, raw payload hash
- Enforce a uniqueness constraint on `(gateway_id, external_event_id)` to dedupe retries.
- Make payment state transitions idempotent (safe to process the same event multiple times).

### PayFast-specific notes (from observed plugin patterns)
- Use `m_payment_id` as your internal lookup key for the payment intent/reservation.
- Verify `amount_gross` matches expected amount.
- Record `pf_payment_id` and treat it as the primary external payment identifier.

---

## Google ops tooling (must-have for running ARI)

### Full sync vs incremental
- Provide a master-admin tool to run a **full sync** per property:
  - Transaction/property data
  - Rates
  - Inventory
  - Availability
- Incremental publishing remains event-driven (outbox).

### Retry / backoff
- Implement exponential backoff with jitter for:
  - HTTP errors
  - Google response errors that indicate temporary issues
- Respect upload limits and batch sizes (avoid >5MB rate files; respect message/sec limits).

### Error dashboard
Per property, show:
- last successful upload time per message type
- last error/warning details
- link to stored payload + Google response

### Persist payloads and responses
- Store:
  - outbound XML payload (or content-addressed hash + blob storage)
  - HTTP status + response body
  - correlation id (`transaction id` in XML)
- This is essential for debugging mismatches and passing Google validation.

---

## Private rates + loyalty (MVP approach)

### Goal
- Support discounted/member/corporate pricing (“private rates”) without building a full CRM.
- Support integration with an existing hotel group loyalty program, while still being able to operate standalone for independent hotels.
- Ensure Google Hotel Ads / Free Booking Links clicks can land on a URL that results in the **same price and eligibility** the user saw on Google.

### Rate plan visibility model
Add a visibility/access layer on top of RatePlan.

- `rate_plan.visibility`:
  - `public`: shown to everyone
  - `private`: only shown if user is eligible
- `rate_plan.private_rate_type` (only if `private`):
  - `loyalty_member` | `corporate` | `promo_code` | `staff` (keep this a small enum)

### Eligibility rules (keep simple)
Represent rate access as a small set of declarative checks:

- `require_login` (boolean)
- `allowed_loyalty_tiers` (array) or `min_tier`
- `allowed_corporate_accounts` (array)
- `required_promo_code` (string)

**Important:** Eligibility is always validated server-side at price quote and at checkout.

### Minimal “CRM” (enough for private rates)
Instead of a marketing CRM, implement **guest identity + memberships**:

- `guest_profile`:
  - email, phone, name
  - marketing consent flags (optional)
- `loyalty_membership`:
  - `provider` (internal | external)
  - `member_id`
  - tier
  - status (active/suspended)
  - linked `guest_profile_id`
- `corporate_account`:
  - name, billing metadata (optional), allowed rate plans

This supports: “member rate”, “corporate negotiated rate”, and “promo-code rate” without building campaigns, pipelines, etc.

### Loyalty program integration (two lanes)

1) **Internal loyalty (fastest MVP)**
- Create simple membership IDs + tiers.
- Provide a lightweight login (email + magic link) to access member rates.

2) **External loyalty connector (hotel groups)**
- Implement a connector interface:
  - `lookupMember(memberId)`
  - `lookupByEmail(email)` (optional)
  - `validateMemberSession(token)`
- Support either:
  - OAuth/OIDC SSO (ideal if the group has it), or
  - signed JWT assertions from the group, or
  - nightly sync import (CSV/API) for members + tiers (simplest operationally).

### Loyalty tier engine (spend-based qualification)

#### Requirement
- Org admins define tiers and the spend required to qualify.
- System calculates each member’s current tier automatically.

#### Definition of “spend” (MVP)
- Spend basis is **room revenue only** (exclude taxes/fees and non-room add-ons) from completed stays.
  - Include: room charges actually paid (net of refunds)
  - Exclude: cancelled reservations, refunded amounts, taxes/fees
- Treat tier calculation as a back-office truth; do not attempt real-time card settlement logic in MVP.

#### Tier rule model (keep it simple)
- `loyalty_program` (per account): name, currency, qualification window.
- `loyalty_tier` (ordered): name, threshold_amount.
- `qualification_window` options:
  - `lifetime` (simplest)
  - `rolling_12_months` (common)

For `rolling_12_months`, the qualifying spend is the sum of ledger amounts with `posted_at >= today_in_property_timezone - 365 days`.
Downgrades are allowed when spend falls below a threshold as the window rolls; recompute at least daily.

**Example:**
- Silver: $0
- Gold: $2,000
- Platinum: $5,000

#### Calculation strategy
- Emit ledger events when money is finalized:
  - `stay.completed` (reservation checked out)
  - `payment.captured` (if you track payments)
  - `refund.issued` (optional)
- Maintain a `loyalty_spend_ledger` (append-only) per member:
  - date, amount, currency, source_reservation_id
- Recompute tier:
  - either incrementally (preferred) on each ledger insert, or
  - nightly job for correctness.

#### Data model additions
- `loyalty_program`:
  - `program_id`, `account_id`, name, currency
  - `qualification_window` (lifetime|rolling_12_months)
- `loyalty_tier`:
  - `tier_id`, `program_id`, name
  - `threshold_amount` (decimal)
  - `sort_order`
- `loyalty_member` (internal provider)
  - `member_id` (string, human-friendly)
  - `guest_profile_id`, `program_id`
  - `current_tier_id`, `qualified_at`
- `loyalty_spend_ledger`
  - `ledger_id`, `program_id`, `guest_profile_id`
  - `posted_at`, `amount`, `currency`, `source`
  - `reservation_id` (nullable)

#### Tier benefits (MVP)
- Map tiers to private rate eligibility:
  - `rate_plan.allowed_loyalty_tiers` uses tier IDs.
- Keep benefits display-only at first (for example “member discount”, “late checkout”) and do not automate operational fulfillment in v1.

### How private rates work with Google clicks (landing page contract)
Your notes include URL variables such as `RATE-RULE-ID`, `CLOSE-RATE-RULE-IDS`, and `USER-LIST-ID`.

**Design principle:** treat the Google click-through as *context* that may indicate a private/conditional rate was selected, but still validate eligibility in your system.

- Landing page accepts parameters (names can be mapped):
  - `rateRuleId` (from Google `RATE-RULE-ID`)
  - `promoCode` (from Google `PROMO-CODE`, if used)
  - optional diagnostics: `closeRateRuleIds`, `userListId`, `verification`

- Server behavior:
  1. Resolve the requested Property + itinerary.
  2. Compute eligible rate plans for the user.
  3. If `rateRuleId`/`promoCode` is present, attempt to map it to an internal `rate_plan_id` (or a rule that selects a plan).
  4. If the mapped plan is not eligible, fall back to public rates and optionally show a friendly “Sign in to unlock member rate” message.
  5. If eligible, **lock the price quote** for a short TTL (for example 10–15 minutes) so the user sees consistent pricing through checkout.

**Note:** The exact mechanics of Google private/conditional rates depend on the program configuration (often Google Ads audience lists / conditional rules). Your system should be able to (a) publish the required rate constructs to Google, and (b) honor the click context when it returns.

---

## Operational requirements (to avoid common Google + booking pitfalls)
- **Stable IDs forever**: property_id / room_type_id / rate_plan_id must never be reused.
- **Atomic booking**: enforce inventory constraints transactionally to prevent overbooking.
- **Timezone correctness**: compute “today” and advance booking window in property timezone.
- **Change-driven publishing**:
  - reservation changes must trigger ARI inventory/availability updates quickly.
  - rate changes must trigger ARI rate updates.
- **Diagnostics**:
  - store last pushed XML payloads (or hashes) + Google responses per message.

---

## Phased delivery (pragmatic)

### Phase 1 (MVP, internal bookings + Google readiness)
- Multi-property admin
- Rooms + RoomTypes + RatePlans
- Reservations by room type with internal room allocation
- Generate Hotel List feed + Landing Pages config

### Phase 2 (Google ARI live)
- ARI publisher worker (Transaction + Rate + Inventory + Availability)
- Feed status dashboard + retry tooling

### Phase 3 (Third-party PMS integrations)
- REST API + webhooks
- Inbound sync modes (external master vs our master)

---

## Open decisions (answers drive implementation details)
1. Will you support Room Bundles / per-room selections in Google, or keep it room-type only?
2. Occupancy pricing: do you need child-age pricing now (requires FOR-EACH-CHILD-AGE variables / extra guest charges)?
3. Who is inventory master per property: your system, or external PMS?

---

## Potential gaps / clarifications (review checklist)

### Booking lifecycle + inventory holds
- Do we support a temporary **hold** while the guest is paying (especially for redirect payments)?
  - Suggested: `hold` expires after TTL; inventory is held during checkout; confirmed after pay-on-arrival booking submit or payment success.
- Modification rules: change dates/room type after confirmation; how to re-price and re-check inventory.

### Pricing model (MVP clarity)
- Define whether totals include taxes/fees and how they’re shown to guests (Google is sensitive to price accuracy).
- Define child pricing inputs:
  - do we collect child ages in the booking UI now, or only `num_children`?
  - if ages are collected, define age bands per property/account.
- Currency support per property (likely one currency per property).

### Policies (guest-facing + Google quality)
- Cancellation policy fields per rate plan (refundable until X days/time, non-refundable, etc.).
- Deposit policy fields for pay-on-arrival (if any) and no-show policy (display-only MVP).

### Payments (beyond PayFast)
- Webhook security and idempotency:
  - signature verification, replay protection, and dedupe by external event id.
- Refunds: even if not automated in MVP, define how manual refunds are recorded for reporting/loyalty ledger.

### Analytics (GTM)
- Prevent double-firing across refresh/back button and cross-domain redirects.
- Decide whether to emit `purchase` for pay-on-arrival immediately (current spec: yes) vs only after check-in (sometimes hotels prefer later). If you keep “on booking confirmed”, align reporting and loyalty ledger accordingly.

### Loyalty engine
- Exact trigger for posting spend:
  - on `checked_out` (current intent) vs on “checked_out AND paid”.
- Guest login UX for member rates (magic link vs password vs SSO for groups) and how this interacts with Google landing URLs.

### Security + compliance (MVP baseline)
- PII handling: encryption at rest for sensitive fields, audit logs for staff actions, data retention.
- API security: scoped API keys, rate limiting, webhook signing.

### Email deliverability + notifications
- Booking confirmation emails, cancellation emails, staff notifications.
- Outbound email provider choice (and SPF/DKIM guidance for org custom domains if you ever want “from: hotel”).

### Google Hotel Center operational tooling
- Initial “full sync” vs incremental pushes; a button to force re-publish property data/rates/inventory.
- Storage of Google upload responses + a dashboard to show feed errors per property.
- Strategy for ARI message batching and backoff when Google rate limits.

### Domain/branding/SEO
- Canonical URL strategy when using custom booking subdomains (avoid duplicate content issues).
- Ability to set logo, favicon, and simple brand tokens; define what is allowed vs fixed.
