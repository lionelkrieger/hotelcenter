## What this repo is (current)

This repository is currently in the **specs-only** phase.

Source of truth:
- `reservation-system-spec.md` (product + integration requirements)
- `vps-setup.md` (Ubuntu VPS setup checklist for MVP hosting)

When implementation begins, this file becomes the contributor/agent guide for:
- how the code is structured (separation of concerns)
- how to run the app locally/on a VPS
- how to debug issues (correlation IDs, stored payloads)

---

## Commands (future, once code exists)

The following commands will be added when we scaffold the Node/TypeScript codebase.

- **Setup:** `npm install`
- **Dev:** `npm run dev` (API + worker + frontend)
- **Build:** `npm run build`
- **Lint:** `npm run lint`
- **Test:** `npm test`

Until the codebase exists, Git operations are the only required commands.

---

## Tech direction (validated)

Recommended for MVP:
- Backend: TypeScript/Node, Postgres, Redis queue
- Frontend: React-based booking UI
- Runtime: Docker Compose on a VPS

Non-negotiables:
- Durable DB transactions (Postgres)
- Background jobs/queue (ARI publishing, webhooks, domain/TLS workflows)
- Idempotent webhook processing
- Structured logs with `correlation_id`

---

## Separation of concerns (required)

This matches the requirements in `reservation-system-spec.md`.

Planned layers:
- **Domain:** pure business rules and invariants
- **Application:** use-cases (create hold, confirm booking, publish ARI, process ITN)
- **Infrastructure:** DB/queue/email/storage + Google/PayFast adapters
- **Interfaces:** HTTP routes/controllers, webhook endpoints, CLI/admin jobs

Rule: API handlers and workers must call the same application services; no duplicated business logic.

---

## Debugging requirements (required)

- All logs are JSON.
- Every request/job has a `correlation_id` and it must propagate into worker jobs.
- Store for replay/debug:
  - inbound webhooks (PayFast ITN)
  - outbound Google ARI payloads + responses

---

## Where to change requirements

- Update product requirements in `reservation-system-spec.md`.
- Update VPS/deploy notes in `vps-setup.md`.

Keep this file high-level. Do not copy requirements, schemas, or workflows here.
Those belong in `reservation-system-spec.md` so there is one source of truth.

If you need to reference a rule (e.g., stable IDs, hold TTL, no double-booking),
link to the relevant section in `reservation-system-spec.md` and summarize in one sentence.

---

## Repository conventions (once implementation starts)

### Code organization
- Use the Domain/Application/Infrastructure/Interfaces separation described in `reservation-system-spec.md`.
- Keep HTTP handlers and workers thin; both call the same application services.

### Data & migrations
- Postgres is the system of record.
- All schema changes happen via migrations (no ad-hoc manual edits).

### Reliability
- Use a durable queue for Google ARI publishing and webhook processing.
- Enforce idempotency in the application layer.

### Observability
- All logs are structured JSON.
- Propagate `correlation_id` across HTTP requests and jobs.

- Minimal business logic (validation only)
- Example: `ReservationController`, `PayFastWebhookHandler`

---

## Configuration Management

### Environment-Driven Configuration

All configuration sourced from environment variables with strict validation at startup.

**Configuration Loading Strategy:**
1. Load from environment variables
2. Apply defaults where appropriate (dev mode only)
3. Validate all required values present
4. Validate value formats (URLs, ports, credentials)
5. Fail fast on startup if config invalid

**Configuration Categories:**

```typescript
// backend/src/infrastructure/config/schema.ts
interface AppConfig {
  // Application
  NODE_ENV: 'development' | 'staging' | 'production';
  PORT: number;
  LOG_LEVEL: 'debug' | 'info' | 'warn' | 'error';
  
  // Database
  DATABASE_URL: string;
  DATABASE_POOL_MIN: number;
  DATABASE_POOL_MAX: number;
  
  // Redis Queue
  REDIS_URL: string;
  REDIS_TLS_ENABLED: boolean;
  
  // Security
  SESSION_SECRET: string;
  API_KEY_SALT: string;
  ENCRYPTION_KEY: string;  // For sensitive data at rest
  
  // Integrations
  GOOGLE_ARI_ENDPOINT: string;
  GOOGLE_ARI_PARTNER_ID: string;
  
  // Email
  EMAIL_PROVIDER: 'sendgrid' | 'ses' | 'smtp';
  EMAIL_FROM_ADDRESS: string;
  EMAIL_API_KEY?: string;
  
  // Object Storage
  STORAGE_PROVIDER: 's3' | 'gcs' | 'local';
  STORAGE_BUCKET: string;
  STORAGE_ACCESS_KEY?: string;
  STORAGE_SECRET_KEY?: string;
  
  // Observability
  SENTRY_DSN?: string;
  METRICS_ENABLED: boolean;
}
```

**Validation Example:**
```typescript
// Validate and fail fast on startup
export function loadConfig(): AppConfig {
  const config = parseEnvironment();
  validateConfig(config);  // Throws if invalid
  return config;
}

// In main.ts
const config = loadConfig();  // Will exit if config invalid
const app = createApp(config);
```

**Per-Tenant Configuration** (stored in database):
- Custom domain settings
- Payment gateway credentials (encrypted)
- Branding assets
- Google Hotel Center linkage
- GTM container IDs

---

## Migration Strategy

### Database Migrations

**Tool:** Use a migration library (e.g., `node-pg-migrate`, `kysely`, or `knex`)

**Migration Naming Convention:**
```
migrations/
  001_initial_schema.sql
  002_add_room_taxonomy.sql
  003_add_loyalty_program.sql
  [timestamp]_[descriptive_name].sql
```

**Migration Structure:**
```sql
-- Up migration
CREATE TABLE properties (
  property_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID NOT NULL REFERENCES accounts(account_id),
  name TEXT NOT NULL,
  timezone TEXT NOT NULL,
  currency TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_properties_account ON properties(account_id);

-- Down migration (in separate section or file)
DROP TABLE IF EXISTS properties;
```

**Migration Best Practices:**
1. **Always reversible:** Provide both up and down migrations
2. **Idempotent:** Use `IF NOT EXISTS`, `IF EXISTS` clauses
3. **Data safety:** Validate data migrations on staging first
4. **Zero-downtime:** Use additive changes, avoid breaking changes
5. **Transactional:** Wrap in transactions where supported

**Migration Execution:**
```bash
# Development
npm run migrate:up          # Apply pending migrations
npm run migrate:down        # Rollback last migration
npm run migrate:create      # Create new migration file

# Production
npm run migrate:up -- --env=production
```

**Schema Versioning:**
- Track applied migrations in `schema_migrations` table
- Include migration hash to detect tampering
- Log migration execution with timestamps

---

## Audit Logging Patterns

### Audit Log Requirements

**What to Audit:**
1. **Staff actions:**
   - Rate/inventory changes (who, what, when)
   - Reservation modifications/cancellations
   - Domain/branding config changes
   - User/permission changes
   
2. **Integration events:**
   - Inbound webhook deliveries (full payload)
   - Outbound Google ARI requests (payload + response)
   - Payment gateway transactions
   
3. **Guest actions:**
   - Reservation creation (for correlation)
   - Cancellations (if self-service enabled)

### Audit Log Schema

```sql
-- Staff audit log
CREATE TABLE audit_log (
  audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID NOT NULL,
  property_id UUID,  -- nullable for account-level actions
  user_id UUID NOT NULL REFERENCES users(user_id),
  action TEXT NOT NULL,  -- e.g., 'rate.update', 'reservation.cancel'
  entity_type TEXT NOT NULL,  -- e.g., 'rate_plan', 'reservation'
  entity_id UUID NOT NULL,
  changes JSONB,  -- old/new values
  metadata JSONB,  -- additional context
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_account_time ON audit_log(account_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);

-- Integration audit log (webhooks, API calls)
CREATE TABLE integration_log (
  log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID NOT NULL,
  property_id UUID,
  integration_type TEXT NOT NULL,  -- 'google_ari', 'payfast_itn', etc.
  direction TEXT NOT NULL,  -- 'inbound', 'outbound'
  request_payload JSONB,
  response_payload JSONB,
  http_status INTEGER,
  correlation_id UUID NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_integration_correlation ON integration_log(correlation_id);
CREATE INDEX idx_integration_account_time ON integration_log(account_id, created_at DESC);
```

### Audit Logging Implementation

**Application-Level Logging:**
```typescript
// backend/src/infrastructure/observability/audit-logger.ts
export interface AuditEntry {
  accountId: string;
  propertyId?: string;
  userId: string;
  action: string;
  entityType: string;
  entityId: string;
  changes?: Record<string, { old: any; new: any }>;
  metadata?: Record<string, any>;
  ipAddress?: string;
  userAgent?: string;
}

export class AuditLogger {
  async log(entry: AuditEntry): Promise<void> {
    await db.query(
      `INSERT INTO audit_log (
        account_id, property_id, user_id, action, 
        entity_type, entity_id, changes, metadata,
        ip_address, user_agent
      ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)`,
      [/* ... */]
    );
  }
}
```

**Usage in Application Layer:**
```typescript
// backend/src/application/commands/update-rate-plan.ts
export class UpdateRatePlanCommand {
  async execute(input: UpdateRatePlanInput, context: CommandContext) {
    const oldRatePlan = await ratePlanRepo.findById(input.ratePlanId);
    const updatedRatePlan = await ratePlanRepo.update(input);
    
    // Audit the change
    await auditLogger.log({
      accountId: context.accountId,
      propertyId: updatedRatePlan.propertyId,
      userId: context.userId,
      action: 'rate_plan.update',
      entityType: 'rate_plan',
      entityId: updatedRatePlan.id,
      changes: calculateChanges(oldRatePlan, updatedRatePlan),
      ipAddress: context.ipAddress,
      userAgent: context.userAgent
    });
    
    return updatedRatePlan;
  }
}
```

**Integration Logging:**
```typescript
// backend/src/infrastructure/integrations/google/ari-publisher.ts
export class GoogleARIPublisher {
  async publishRates(payload: ARIPayload): Promise<void> {
    const correlationId = generateUUID();
    
    try {
      const response = await httpClient.post(endpoint, payload, {
        headers: { 'X-Correlation-ID': correlationId }
      });
      
      // Log success
      await integrationLogger.log({
        accountId: payload.accountId,
        propertyId: payload.propertyId,
        integrationType: 'google_ari',
        direction: 'outbound',
        requestPayload: payload,
        responsePayload: response.data,
        httpStatus: response.status,
        correlationId
      });
    } catch (error) {
      // Log failure
      await integrationLogger.log({
        accountId: payload.accountId,
        propertyId: payload.propertyId,
        integrationType: 'google_ari',
        direction: 'outbound',
        requestPayload: payload,
        responsePayload: { error: error.message },
        httpStatus: error.response?.status || 0,
        correlationId
      });
      
      throw error;
    }
  }
}
```

### Audit Log Retention

- **Staff audit log:** Retain 7 years (compliance)
- **Integration log:** Retain 90 days (debugging), then archive
- **High-volume logs:** Consider partitioning by month

---

## Structured Logging

### Logging Format

All logs output as JSON for machine parsing:

```json
{
  "timestamp": "2025-12-30T14:23:45.123Z",
  "level": "info",
  "message": "Reservation created",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
  "account_id": "acc_123",
  "property_id": "prop_456",
  "user_id": "usr_789",
  "reservation_id": "res_abc",
  "context": {
    "checkin_date": "2025-12-25",
    "checkout_date": "2025-12-27"
  }
}
```

### Correlation ID Propagation

**Generation:**
- HTTP requests: Extract from `X-Correlation-ID` header or generate new UUID
- Background jobs: Inherit from triggering request or generate new

**Propagation:**
- Store in request context (AsyncLocalStorage or similar)
- Pass to all log calls within request scope
- Include in outbound HTTP requests
- Store in background job metadata

**Implementation:**
```typescript
// backend/src/infrastructure/observability/logger.ts
export class Logger {
  private correlationId: string;
  
  constructor(correlationId: string) {
    this.correlationId = correlationId;
  }
  
  info(message: string, context?: Record<string, any>) {
    console.log(JSON.stringify({
      timestamp: new Date().toISOString(),
      level: 'info',
      message,
      correlation_id: this.correlationId,
      ...context
    }));
  }
}

// HTTP middleware
app.use((req, res, next) => {
  const correlationId = req.headers['x-correlation-id'] || generateUUID();
  req.correlationId = correlationId;
  res.setHeader('X-Correlation-ID', correlationId);
  req.logger = new Logger(correlationId);
  next();
});
```

---

## Phased Delivery Roadmap

### Phase 1: MVP Internal Bookings (Weeks 1-8)

**Goal:** Enable internal staff to create reservations; establish foundation for Google integration.

**Deliverables:**
- [ ] Multi-tenant data model (accounts, properties, users, permissions)
- [ ] Staff authentication (magic links)
- [ ] Property management (CRUD properties, room types, rooms)
- [ ] Room taxonomy (amenities, views, bed config, accessibility)
- [ ] Rate plan management (base rates, derived rates, restrictions)
- [ ] Availability calculation engine
- [ ] Pricing engine (base + modifiers, occupancy-based)
- [ ] Reservation creation (internal staff tool)
- [ ] Room allocation (assign physical rooms to bookings)
- [ ] Basic admin dashboard (reservations, inventory, rates)
- [ ] Database schema + migrations
- [ ] Structured logging infrastructure
- [ ] Configuration validation
- [ ] Audit logging for staff actions

**Out of Scope (Phase 1):**
- Guest-facing booking UI
- Payment processing
- Google ARI publishing
- Background workers
- Custom domain setup

**Success Criteria:**
- Staff can create test reservations
- Availability correctly decrements inventory
- Rate plans calculate prices correctly
- All staff actions audited

---

### Phase 2: Google ARI Live (Weeks 9-14)

**Goal:** Go live with Google Hotel Ads / Free Booking Links.

**Deliverables:**
- [ ] Guest-facing booking UI (search, room selection, checkout)
- [ ] Guest booking subdomain (default: `book.yoursaas.com/org-slug`)
- [ ] Google Hotel List feed generator
- [ ] Google Landing Pages / Point of Sale configuration
- [ ] Google ARI publisher (Transaction, Rate, Inventory, Availability)
- [ ] Background queue infrastructure (Redis-backed)
- [ ] ARI worker with retry/backoff logic
- [ ] Event-driven ARI publishing (on inventory/rate changes)
- [ ] Integration logging (store payloads + responses)
- [ ] Google ops dashboard (feed status, last sync, errors)
- [ ] Manual full-sync tool (admin)
- [ ] Payment support: Pay on Arrival (no gateway)
- [ ] Booking confirmation emails
- [ ] Hold expiry worker (release inventory after TTL)
- [ ] Google Analytics / GTM integration (ecommerce events)

**Out of Scope (Phase 2):**
- PayFast integration
- Custom CNAME domains
- Loyalty program
- Private/member rates
- Third-party API

**Success Criteria:**
- Properties visible on Google Hotel Ads
- Prices and availability match between Google and booking UI
- Reservations flow end-to-end (search → book → confirm)
- ARI updates pushed within 5 minutes of inventory changes
- All integration payloads logged for debugging

---

### Phase 3: PMS Integrations (Weeks 15-20)

**Goal:** Enable hotel groups to integrate with their existing PMS/channel manager systems.

**Deliverables:**
- [ ] REST API (versioned `/api/v1`)
  - [ ] Properties, room types, rate plans (read)
  - [ ] Availability queries
  - [ ] Create/modify/cancel reservations
  - [ ] Push rates bulk endpoint
  - [ ] Push inventory bulk endpoint
  - [ ] Push restrictions bulk endpoint
- [ ] Webhook delivery system
  - [ ] `reservation.created`, `reservation.modified`, `reservation.cancelled`
  - [ ] `inventory.changed`, `rate.changed`
- [ ] API key management (scoped per property/integration)
- [ ] API documentation (OpenAPI/Swagger)
- [ ] Webhook signature verification
- [ ] Idempotency keys for write operations
- [ ] External inventory master mode (flag per property)
- [ ] Sync conflict resolution (timestamp-based)
- [ ] Rate limiting per API key
- [ ] Integration health monitoring

**Out of Scope (Phase 3):**
- Pre-built PMS connectors (custom per integration)
- Real-time two-way sync (event-driven push model)

**Success Criteria:**
- External system can push rates and receive reservations
- Webhooks deliver reliably with retries
- API key permissions enforced correctly
- No data loss or race conditions in dual-write scenarios

---

### Phase 4: Advanced Features (Post-MVP)

**Future Enhancements (prioritize based on customer feedback):**
- [ ] PayFast integration (prepayment flow, ITN webhook)
- [ ] Extensible payment gateway framework
- [ ] Custom domain setup (CNAME verification, TLS provisioning)
- [ ] Loyalty program (internal tiers, spend tracking)
- [ ] Private/member rates (tiered discounts)
- [ ] Promo codes
- [ ] Corporate negotiated rates
- [ ] External loyalty SSO (OAuth/JWT)
- [ ] Branding customization (logo, banner, colors)
- [ ] Multi-language support
- [ ] Multi-currency support
- [ ] Advanced child pricing (age bands)
- [ ] Guest self-service (modify/cancel bookings)
- [ ] Refund workflows
- [ ] Housekeeping basic workflows
- [ ] Reporting dashboard (revenue, occupancy, ADR)
- [ ] Email marketing (abandoned cart recovery)
- [ ] Live on Google API management

---

## Operational Runbook

### Tracing a Booking End-to-End

**Using correlation_id:**
```bash
# Search logs by correlation_id
grep '"correlation_id":"550e8400-e29b-41d4-a716-446655440000"' /var/log/app.log

# Query audit log
SELECT * FROM audit_log WHERE correlation_id = '550e8400-e29b-41d4-a716-446655440000' ORDER BY created_at;

# Query integration log
SELECT * FROM integration_log WHERE correlation_id = '550e8400-e29b-41d4-a716-446655440000' ORDER BY created_at;
```

**Using reservation_id:**
```sql
-- Find reservation and related events
SELECT * FROM reservations WHERE reservation_id = 'res_abc';
SELECT * FROM audit_log WHERE entity_id = 'res_abc' ORDER BY created_at;
SELECT * FROM integration_log WHERE request_payload->>'reservation_id' = 'res_abc' ORDER BY created_at;
```

### Re-driving ARI Publishing

**For a specific property and date range:**
```bash
# Via admin CLI
npm run cli -- google:publish-ari \
  --property-id=prop_123 \
  --start-date=2025-12-25 \
  --end-date=2026-01-10 \
  --force

# Via admin dashboard
# Navigate to: Admin > Properties > [Property] > Google ARI > Manual Sync
```

**For all properties (full sync):**
```bash
npm run cli -- google:full-sync --all-properties
```

### Inspecting Webhook History

**For a reservation:**
```sql
-- Find webhook deliveries related to reservation
SELECT * FROM integration_log
WHERE integration_type = 'payfast_itn'
  AND request_payload->>'m_payment_id' = 'res_abc'
ORDER BY created_at DESC;
```

**For a payment:**
```sql
-- Find payment gateway webhook
SELECT * FROM integration_log
WHERE integration_type = 'payfast_itn'
  AND request_payload->>'pf_payment_id' = '12345'
ORDER BY created_at DESC;
```

### Monitoring ARI Health

**Dashboard queries:**
```sql
-- Last successful ARI push per property
SELECT 
  property_id,
  MAX(created_at) as last_success,
  NOW() - MAX(created_at) as time_since_last
FROM integration_log
WHERE integration_type = 'google_ari'
  AND direction = 'outbound'
  AND http_status = 200
GROUP BY property_id
HAVING NOW() - MAX(created_at) > INTERVAL '1 hour';  -- Alert if stale

-- Recent ARI errors
SELECT 
  property_id,
  COUNT(*) as error_count,
  MAX(created_at) as last_error,
  response_payload->>'error' as error_message
FROM integration_log
WHERE integration_type = 'google_ari'
  AND direction = 'outbound'
  AND http_status != 200
  AND created_at > NOW() - INTERVAL '24 hours'
GROUP BY property_id, response_payload->>'error'
ORDER BY error_count DESC;
```

---

## Multi-Tenant Security

### Tenant Isolation

**Data Scoping:**
- Every database row includes `account_id` and/or `property_id`
- All queries MUST filter by tenant scope
- Use row-level security policies (Postgres RLS) as defense-in-depth

**Routing:**
- HTTP requests routed by `Host` header → `custom_domain` → `account_id`
- API requests authenticated via API key → `account_id`
- Background jobs inherit `account_id` from triggering event

**Repository Pattern:**
```typescript
// Always scope queries by account
export class ReservationRepository {
  async findById(accountId: string, reservationId: string): Promise<Reservation> {
    return db.query(
      'SELECT * FROM reservations WHERE account_id = $1 AND reservation_id = $2',
      [accountId, reservationId]
    );
  }
}
```

### Permission Enforcement

**Middleware:**
```typescript
// Check permissions on each request
app.use(requirePermission('reservations.write'));

function requirePermission(permission: string) {
  return async (req, res, next) => {
    const hasPermission = await permissionService.check(
      req.user.userId,
      req.account.accountId,
      permission
    );
    if (!hasPermission) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}
```

---

## Testing Strategy

**Unit Tests:**
- Domain logic (entities, value objects, services)
- Application services (commands, queries)
- Pure functions and utilities
- Target: 80%+ coverage

**Integration Tests:**
- Repository implementations (with test database)
- External service adapters (with mocks/stubs)
- Queue workers
- Target: Critical paths covered

**End-to-End Tests:**
- Guest booking flow (search → checkout → confirmation)
- Staff reservation management
- ARI publishing (with Google sandbox)
- Webhook processing (PayFast sandbox)
- Target: Happy paths + key error cases

**Test Database:**
- Use Docker container for local tests
- Reset schema between test runs
- Seed with minimal fixture data

---

## Security Checklist

- [ ] All secrets in environment variables (never committed)
- [ ] Secrets encrypted at rest in database (payment credentials)
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS prevention (React default escaping + CSP headers)
- [ ] CSRF protection (token-based for state-changing operations)
- [ ] Rate limiting (per IP, per API key)
- [ ] Webhook signature verification (all inbound webhooks)
- [ ] TLS enforced (HTTPS only, HSTS headers)
- [ ] Audit logs for all sensitive operations
- [ ] Permission checks on all authenticated endpoints
- [ ] API keys scoped to specific properties
- [ ] Magic link tokens single-use and short-lived
- [ ] Session tokens secure, HttpOnly, SameSite
- [ ] PII handling (minimize collection, encrypt where needed)
- [ ] GDPR compliance (data export, deletion on request)
