# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Cleaning Services Platform · Created: 2026-05-22

## Philosophy

This model treats every state change as an immutable event appended to a central event store. The current state of any entity — a job, a cleaner's schedule, an inspection — is derived by replaying its event history. Read-optimised projections (materialised views) are maintained separately from the write path, following the Command Query Responsibility Segregation (CQRS) pattern.

The motivation is the cleaning industry's growing demand for full audit trails. Commercial cleaning contracts increasingly require proof that specific cleaners were on-site at specific times, that inspections were conducted with specific results, and that compliance-relevant products were used. Healthcare facilities under HIPAA need immutable records of cleaning activities. CIMS certification requires documented evidence of quality systems. An event-sourced architecture provides all of this by design: every action is recorded, timestamped, and immutable. "What was the state of this contract on January 15th?" is a query you can answer by replaying events to that point.

The architecture separates the write side (commands that produce events) from the read side (projections optimised for queries). This enables independent scaling — the write path can be optimised for append-only throughput while the read path uses materialised views tailored to each UI screen. The trade-off is significantly higher architectural complexity: developers must think in events rather than CRUD, and eventual consistency between the event store and projections must be managed carefully.

**Best for:** Organisations that need irrefutable audit trails, temporal queries ("what was true on date X?"), compliance evidence for CIMS/HIPAA/LEED, and AI analytics on historical change patterns.

**Trade-offs:**
- (+) Complete, immutable audit trail — every change is recorded forever
- (+) Temporal queries: reconstruct any entity's state at any point in time
- (+) Natural fit for compliance evidence (CIMS, HIPAA, OSHA)
- (+) Events feed AI analytics pipelines directly (cleaner retention prediction, pricing optimisation)
- (+) Read models can be rebuilt from scratch if corrupted or if a new projection is needed
- (-) Significantly higher architectural complexity — event replay, projection rebuilds, eventual consistency
- (-) Developers must think in events, not CRUD — steeper learning curve
- (-) Projection lag means the read side may be milliseconds to seconds behind the write side
- (-) Schema evolution of events requires careful versioning (upcasting)
- (-) Debugging is harder — current state is computed, not stored directly

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISSA CIMS | Every compliance-relevant action (inspection, training, procedure update) is an event with full provenance — natural evidence for CIMS auditors |
| OSHA HazCom | Chemical usage events are immutable; SDS assignment changes are tracked with before/after state |
| HIPAA | Healthcare facility cleaning events are immutable and cannot be altered after recording — supports HIPAA audit requirements |
| EPA Safer Choice / LEED | Product compliance status changes are events; auditors can verify the exact compliance state at any historical date |
| ISO 9001 | Event history provides the "documented quality management system" evidence ISO 9001 requires |
| RFC 5545 | Schedule change events capture recurrence rule modifications with full before/after state |
| AsyncAPI 2.x | Event contracts are documented using AsyncAPI, defining the structure of each event type for consumers |

---

## Event Store

```sql
-- The single source of truth: an append-only event log
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,   -- job, client, cleaner, inspection, invoice, contract, chemical, equipment
    stream_id       UUID NOT NULL,           -- the aggregate root ID
    event_type      VARCHAR(100) NOT NULL,   -- e.g. job.created, job.assigned, job.clock_in, inspection.completed
    event_version   INTEGER NOT NULL,        -- sequential within stream; optimistic concurrency
    payload         JSONB NOT NULL,           -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- causation_id, correlation_id, user_id, ip_address
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Enforce ordering within a stream
    UNIQUE (stream_id, event_version)
);

-- Primary query patterns
CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_tenant_type ON event_store(tenant_id, stream_type, created_at);
CREATE INDEX idx_event_type ON event_store(event_type, created_at);
CREATE INDEX idx_event_created ON event_store(created_at);

-- GIN index for querying event payloads
CREATE INDEX idx_event_payload ON event_store USING GIN (payload jsonb_path_ops);
```

### Event Type Catalogue

```
-- Job lifecycle events
job.created              -- {client_id, property_id, service_type_id, scheduled_start, scheduled_end, quoted_price}
job.updated              -- {changes: {field: {old, new}}}
job.assigned             -- {cleaner_id, role}
job.unassigned           -- {cleaner_id, reason}
job.dispatched           -- {route_id, estimated_arrival}
job.clock_in             -- {cleaner_id, latitude, longitude, accuracy, within_geofence}
job.clock_out            -- {cleaner_id, latitude, longitude, duration_minutes}
job.checklist_completed  -- {item_id, completed_by}
job.photo_uploaded       -- {photo_id, storage_key, photo_type, latitude, longitude}
job.completed            -- {final_price, duration_minutes}
job.cancelled            -- {reason, cancelled_by}
job.no_show              -- {reason}

-- Client lifecycle events
client.created           -- {name, email, phone, source}
client.updated           -- {changes: {field: {old, new}}}
client.archived          -- {reason}

-- Cleaner lifecycle events
cleaner.hired            -- {first_name, last_name, employment_type, hourly_rate}
cleaner.skill_added      -- {skill_id, proficiency}
cleaner.availability_set -- {day_of_week, start_time, end_time}
cleaner.deactivated      -- {reason}

-- Inspection events
inspection.started       -- {template_id, property_id, inspector_id, latitude, longitude}
inspection.item_rated    -- {template_item_id, rating, numeric_score, notes}
inspection.photo_added   -- {result_id, storage_key, caption}
inspection.completed     -- {overall_score}
inspection.report_sent   -- {recipient_email, report_storage_key}

-- Invoice events
invoice.created          -- {client_id, line_items: [...]}
invoice.sent             -- {recipient_email}
invoice.viewed           -- {viewed_at}
invoice.paid             -- {payment_id, amount, method}
invoice.voided           -- {reason}

-- Schedule events
schedule.rule_created    -- {client_id, property_id, frequency, day_of_week, start_time}
schedule.rule_updated    -- {changes: {field: {old, new}}}
schedule.rule_paused     -- {reason}
schedule.rule_cancelled  -- {reason}
schedule.jobs_generated  -- {job_ids: [...], period_start, period_end}

-- Compliance events
compliance.product_added        -- {product_id, name, cas_number, certifications}
compliance.product_used         -- {job_id, product_id, quantity, unit, cleaner_id}
compliance.sds_uploaded         -- {product_id, storage_key, revision_date}
compliance.cims_element_updated -- {element_id, status, evidence_notes}
compliance.certification_added  -- {product_id, certification, valid_from, valid_until}

-- Contract events (commercial)
contract.created         -- {client_id, properties, services, monthly_value, terms}
contract.activated       -- {start_date}
contract.amended         -- {changes: {field: {old, new}}}
contract.suspended       -- {reason}
contract.terminated      -- {reason, effective_date}

-- Communication events
communication.sent       -- {channel, recipient_type, recipient_id, template, external_id}
communication.delivered  -- {external_id}
communication.failed     -- {external_id, error}
communication.bounced    -- {external_id, reason}

-- Route events
route.optimised          -- {cleaner_id, date, stops: [...], total_distance_km, total_duration_minutes}
route.stop_reordered     -- {old_order, new_order, reason}
route.recalculated       -- {trigger, new_stops: [...]}
```

## Read-Side Projections (Materialised Views)

These tables are rebuilt from events. They are the query targets for the application.

### Projection: Current Job State

```sql
CREATE TABLE projection_job (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    job_number      INTEGER,
    client_id       UUID NOT NULL,
    client_name     VARCHAR(255),
    property_id     UUID NOT NULL,
    property_address VARCHAR(500),
    service_type_id UUID NOT NULL,
    service_type_name VARCHAR(100),
    schedule_rule_id UUID,
    status          VARCHAR(30) NOT NULL,
    priority        VARCHAR(10) NOT NULL DEFAULT 'normal',
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end   TIMESTAMPTZ NOT NULL,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    duration_minutes INTEGER,
    quoted_price    DECIMAL(10, 2),
    final_price     DECIMAL(10, 2),
    instructions    TEXT,
    assigned_cleaners JSONB DEFAULT '[]',
    -- Example: [{"cleaner_id": "uuid", "name": "Maria G.", "role": "lead", "status": "accepted"}]
    checklist_progress JSONB DEFAULT '{}',
    -- Example: {"total": 12, "completed": 8, "percent": 66.7}
    photo_count     INTEGER DEFAULT 0,
    last_event_at   TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_job_tenant ON projection_job(tenant_id);
CREATE INDEX idx_proj_job_schedule ON projection_job(tenant_id, scheduled_start);
CREATE INDEX idx_proj_job_status ON projection_job(tenant_id, status);
CREATE INDEX idx_proj_job_client ON projection_job(client_id);
```

### Projection: Current Client State

```sql
CREATE TABLE projection_client (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    is_company      BOOLEAN NOT NULL DEFAULT false,
    company_name    VARCHAR(255),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    email           VARCHAR(255),
    phone           VARCHAR(30),
    billing_address JSONB,
    source          VARCHAR(50),
    tags            TEXT[],
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    -- Denormalized aggregates
    property_count  INTEGER DEFAULT 0,
    total_jobs      INTEGER DEFAULT 0,
    total_revenue   DECIMAL(12, 2) DEFAULT 0,
    outstanding_balance DECIMAL(10, 2) DEFAULT 0,
    last_job_date   DATE,
    next_job_date   DATE,
    average_rating  DECIMAL(3, 2),
    last_event_at   TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_client_tenant ON projection_client(tenant_id);
CREATE INDEX idx_proj_client_name ON projection_client(tenant_id, display_name);
```

### Projection: Current Cleaner State

```sql
CREATE TABLE projection_cleaner (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(30),
    employment_type VARCHAR(20),
    hourly_rate     DECIMAL(10, 2),
    preferred_locale VARCHAR(10) DEFAULT 'en',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    hire_date       DATE,
    home_location   JSONB,
    -- Example: {"latitude": 40.7128, "longitude": -74.0060, "city": "New York"}
    skills          JSONB DEFAULT '[]',
    -- Example: [{"skill_id": "uuid", "name": "deep_clean", "proficiency": "expert"}]
    availability    JSONB DEFAULT '[]',
    -- Example: [{"day": 1, "start": "08:00", "end": "17:00"}]
    -- Denormalized performance
    total_jobs_completed INTEGER DEFAULT 0,
    avg_job_duration_minutes DECIMAL(8, 1),
    average_inspection_score DECIMAL(5, 2),
    hours_this_week DECIMAL(6, 1) DEFAULT 0,
    hours_this_month DECIMAL(8, 1) DEFAULT 0,
    last_event_at   TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_cleaner_tenant ON projection_cleaner(tenant_id);
CREATE INDEX idx_proj_cleaner_active ON projection_cleaner(tenant_id, is_active);
```

### Projection: Current Inspection State

```sql
CREATE TABLE projection_inspection (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    job_id          UUID,
    property_id     UUID NOT NULL,
    property_name   VARCHAR(255),
    template_id     UUID NOT NULL,
    template_name   VARCHAR(255),
    inspector_id    UUID NOT NULL,
    inspector_name  VARCHAR(255),
    status          VARCHAR(20) NOT NULL,
    overall_score   DECIMAL(5, 2),
    items_total     INTEGER DEFAULT 0,
    items_completed INTEGER DEFAULT 0,
    items_passed    INTEGER DEFAULT 0,
    items_failed    INTEGER DEFAULT 0,
    photo_count     INTEGER DEFAULT 0,
    location        JSONB,
    inspected_at    TIMESTAMPTZ,
    submitted_at    TIMESTAMPTZ,
    report_sent     BOOLEAN DEFAULT false,
    last_event_at   TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_inspection_tenant ON projection_inspection(tenant_id);
CREATE INDEX idx_proj_inspection_property ON projection_inspection(property_id);
```

### Projection: Compliance Dashboard

```sql
CREATE TABLE projection_compliance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    -- CIMS status
    cims_elements_total     INTEGER DEFAULT 0,
    cims_mandatory_compliant INTEGER DEFAULT 0,
    cims_mandatory_total    INTEGER DEFAULT 0,
    cims_recommended_compliant INTEGER DEFAULT 0,
    cims_recommended_total  INTEGER DEFAULT 0,
    cims_ready_for_certification BOOLEAN DEFAULT false,
    -- Chemical compliance
    total_products          INTEGER DEFAULT 0,
    products_with_sds       INTEGER DEFAULT 0,
    products_epa_safer_choice INTEGER DEFAULT 0,
    products_leed_compliant INTEGER DEFAULT 0,
    sds_expiring_30_days    INTEGER DEFAULT 0,
    -- Per-section breakdown
    section_scores          JSONB DEFAULT '{}',
    -- Example: {"QS": {"compliant": 8, "total": 10}, "HR": {"compliant": 5, "total": 7}}
    last_recalculated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id)
);
```

### Projection: Schedule Calendar

```sql
CREATE TABLE projection_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    job_id          UUID NOT NULL,
    cleaner_id      UUID,
    cleaner_name    VARCHAR(255),
    client_name     VARCHAR(255),
    property_address VARCHAR(500),
    service_type    VARCHAR(100),
    status          VARCHAR(30) NOT NULL,
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end   TIMESTAMPTZ NOT NULL,
    route_id        UUID,
    stop_order      SMALLINT,
    last_event_at   TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_schedule_tenant ON projection_schedule(tenant_id, scheduled_start);
CREATE INDEX idx_proj_schedule_cleaner ON projection_schedule(cleaner_id, scheduled_start);
```

### Projection: Route Map

```sql
CREATE TABLE projection_route (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    cleaner_id      UUID NOT NULL,
    cleaner_name    VARCHAR(255),
    route_date      DATE NOT NULL,
    is_optimised    BOOLEAN DEFAULT false,
    total_distance_km DECIMAL(8, 2),
    total_duration_minutes INTEGER,
    stops           JSONB DEFAULT '[]',
    -- Example: [{"stop_order": 1, "job_id": "uuid", "address": "123 Main St", 
    --            "estimated_arrival": "2026-05-22T09:00:00Z", "status": "pending"}]
    last_event_at   TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_route_tenant ON projection_route(tenant_id, route_date);
CREATE INDEX idx_proj_route_cleaner ON projection_route(cleaner_id, route_date);
```

## Reference Data (Shared, Non-Event-Sourced)

```sql
-- These rarely-changing lookup tables are not event-sourced
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    phone           VARCHAR(30),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    locale          VARCHAR(10) NOT NULL DEFAULT 'en',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE service_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    default_duration_minutes INTEGER NOT NULL DEFAULT 120,
    base_price      DECIMAL(10, 2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (tenant_id, name)
);

CREATE TABLE inspection_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    sections        JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"name": "Kitchen", "weight": 1.0, "items": [
    --   {"description": "Countertops wiped", "rating_type": "pass_fail", "required": true}
    -- ]}]
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE skill (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    UNIQUE (tenant_id, name)
);

CREATE TABLE cims_section (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(10) NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL,
    description     TEXT
);

CREATE TABLE cims_element (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id      UUID NOT NULL REFERENCES cims_section(id),
    code            VARCHAR(20) NOT NULL UNIQUE,
    title           VARCHAR(255) NOT NULL,
    is_mandatory    BOOLEAN NOT NULL DEFAULT false,
    sort_order      SMALLINT NOT NULL DEFAULT 0
);
```

## Snapshot Store (Performance Optimisation)

```sql
-- Snapshots avoid replaying long event histories for busy aggregates
CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,  -- event_version at snapshot time
    state           JSONB NOT NULL,     -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- Keep only the latest snapshot per stream (older ones can be pruned)
CREATE INDEX idx_snapshot_latest ON event_snapshot(stream_id, snapshot_version DESC);
```

## Example: Replaying a Job's History

```sql
-- Get the complete history of a job
SELECT event_type, payload, metadata, created_at
FROM event_store
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND stream_type = 'job'
ORDER BY event_version ASC;

-- Result (example):
-- event_type          | payload                                          | created_at
-- job.created         | {"client_id":"...","scheduled_start":"2026-05-22T09:00:00Z",...} | 2026-05-20T14:30:00Z
-- job.assigned        | {"cleaner_id":"...","role":"lead"}                | 2026-05-20T14:32:00Z
-- job.dispatched      | {"route_id":"...","estimated_arrival":"09:05"}    | 2026-05-22T07:00:00Z
-- job.clock_in        | {"cleaner_id":"...","lat":40.7128,"lng":-74.006} | 2026-05-22T09:03:00Z
-- job.checklist_completed | {"item_id":"...","completed_by":"..."}       | 2026-05-22T09:45:00Z
-- job.photo_uploaded  | {"storage_key":"s3://...","photo_type":"after"}   | 2026-05-22T11:15:00Z
-- job.clock_out       | {"cleaner_id":"...","lat":40.7128,"lng":-74.006} | 2026-05-22T11:20:00Z
-- job.completed       | {"final_price":150.00,"duration_minutes":137}     | 2026-05-22T11:20:00Z
```

## Example: Time-Travel Query

```sql
-- What was the state of a cleaner's skills on January 15, 2026?
SELECT payload
FROM event_store
WHERE stream_id = 'cleaner-uuid-here'
  AND stream_type = 'cleaner'
  AND event_type IN ('cleaner.hired', 'cleaner.skill_added', 'cleaner.skill_removed')
  AND created_at <= '2026-01-15T23:59:59Z'
ORDER BY event_version ASC;
-- Application code replays these events to reconstruct the skill set at that date
```

## Example: Compliance Audit Query

```sql
-- All chemical usage events for a LEED-certified property in Q1 2026
SELECT e.event_type, e.payload, e.created_at
FROM event_store e
WHERE e.tenant_id = 'tenant-uuid'
  AND e.event_type = 'compliance.product_used'
  AND e.payload->>'property_id' IN (
      SELECT id::text FROM projection_job WHERE property_id IN (
          SELECT id FROM projection_inspection WHERE property_id = 'leed-property-uuid'
      )
  )
  AND e.created_at BETWEEN '2026-01-01' AND '2026-03-31'
ORDER BY e.created_at;
```

## Projection Rebuild Infrastructure

```sql
-- Track projection rebuild state
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    rebuild_started_at TIMESTAMPTZ,
    rebuild_completed_at TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'live'  -- live, rebuilding, failed
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single append-only event_store table — the source of truth |
| Snapshots | 1 | event_snapshot for replay performance |
| Reference Data | 7 | tenant, user, service_type, inspection_template, skill, cims_section, cims_element |
| Projections | 7 | projection_job, projection_client, projection_cleaner, projection_inspection, projection_compliance, projection_schedule, projection_route |
| Infrastructure | 1 | projection_checkpoint |
| **Total** | **~17** | Far fewer tables than normalized; complexity lives in event types and projection logic |

---

## Key Design Decisions

1. **Single event_store table with stream_type/stream_id** — all events for all aggregate types live in one table, partitioned logically by stream. This simplifies infrastructure (one table to back up, replicate, and monitor) and enables cross-aggregate queries. For very high volumes, PostgreSQL table partitioning by `created_at` (monthly) is recommended.

2. **JSONB payloads with GIN indexing** — event payloads are stored as JSONB for schema flexibility. Each event type has a well-defined JSON Schema (documented via AsyncAPI), but new fields can be added without table migrations. The GIN index enables querying into payloads when needed.

3. **Optimistic concurrency via event_version** — the UNIQUE constraint on `(stream_id, event_version)` prevents conflicting writes. If two commands try to append events to the same stream simultaneously, one will fail and must retry with the updated version.

4. **Projections are disposable and rebuildable** — every projection table can be dropped and rebuilt from the event store. This means new projections can be added at any time (e.g., a new analytics dashboard) without touching the write side.

5. **Snapshots for long-lived aggregates** — a job with hundreds of events (frequent check-ins, photo uploads, checklist items) would be slow to replay from scratch. Periodic snapshots capture the aggregate state at a point, so replay only needs events after the snapshot.

6. **Reference data tables are not event-sourced** — tenant, user, service_type, and skill tables are low-churn reference data. Event-sourcing them would add complexity without meaningful benefit. They use standard CRUD.

7. **Inspection templates stored as JSONB in reference data** — since templates are versioned documents referenced by inspection events, storing the full template structure as JSONB simplifies versioning. The event `inspection.started` captures the template_id at the time of inspection, preserving point-in-time accuracy.

8. **Rich metadata on every event** — the `metadata` JSONB field captures causation_id (what caused this event), correlation_id (request trace), user_id, and IP address. This supports debugging, security audit, and compliance investigations.

9. **Events feed AI pipelines directly** — the event store is the natural source for streaming to analytics systems. Cleaner retention prediction, dynamic pricing, and quality trend analysis all consume the event stream without needing to query the relational projections.

10. **Eventual consistency is explicit** — the application acknowledges that projections may lag behind events by milliseconds to seconds. The UI can use the write side's response (with the new event) for immediate feedback while projections update asynchronously.
