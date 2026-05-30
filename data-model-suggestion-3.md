# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Cleaning Services Platform · Created: 2026-05-22

## Philosophy

This model uses a relational core for the entities that are universal across all cleaning businesses — clients, properties, jobs, cleaners, invoices — with JSONB columns for the parts that vary by business type, jurisdiction, service offering, or client. The idea is that a residential maid service in Austin, a commercial janitorial company in Toronto, and a healthcare cleaning contractor in London all share the same scheduling and billing core, but their inspection checklists, compliance fields, contract terms, and property metadata differ dramatically.

Rather than modelling every possible variation as its own column or table (which leads to sparse, wide tables or an explosion of junction tables), the hybrid approach puts stable, query-critical fields in typed relational columns and pushes variable, context-dependent fields into JSONB. PostgreSQL's JSONB support is mature: GIN indexes enable efficient containment queries, JSON Schema validation can enforce structure at the application layer, and JSONB columns participate in standard SQL queries seamlessly.

This is the pattern used by many modern SaaS platforms that serve diverse customer segments from a single codebase. It enables rapid iteration (adding a new field to a JSONB column requires no migration) while preserving the query performance and integrity of relational columns for the data that matters most.

**Best for:** Teams building a multi-market platform (residential + commercial + healthcare) where property metadata, compliance requirements, and service configurations vary widely across customer segments, and rapid iteration speed matters more than rigid schema enforcement.

**Trade-offs:**
- (+) Fastest time-to-market — new fields require no migrations, just application code changes
- (+) Naturally handles residential/commercial/healthcare variation without separate schemas
- (+) Fewer tables than fully normalised (~30 vs ~46) — simpler to understand and maintain
- (+) JSONB columns are self-documenting with JSON Schema definitions
- (+) PostgreSQL GIN indexes provide efficient queries into JSONB
- (-) JSONB fields are not enforced at the database level — validation must happen in application code
- (-) Reporting on JSONB fields requires extraction (`->>`), which is slower than column queries
- (-) Schema evolution of JSONB structures must be managed carefully to avoid data drift
- (-) Developers must know which fields are relational and which are JSONB — mixed patterns can confuse

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISSA CIMS | CIMS compliance tracked via `compliance_records` JSONB on tenant, allowing different CIMS versions and custom elements per organisation |
| OSHA HazCom | Chemical product `certifications` and `hazard_info` stored as JSONB — accommodates varying SDS fields across manufacturers |
| EPA Safer Choice / LEED | Property-level `compliance_requirements` JSONB specifies which certifications are needed; product `certifications` JSONB tracks which are held |
| ISO 3166 | Country/subdivision codes in relational columns; jurisdiction-specific property fields in JSONB |
| RFC 5545 | Recurrence rules stored as JSONB matching RRULE structure: `{"freq": "WEEKLY", "interval": 1, "byday": ["MO","TH"]}` |
| JSON Schema (Draft 2020-12) | Every JSONB column has a documented JSON Schema definition; application layer validates on write |
| GS1 GLN | Optional field in property `metadata` JSONB for commercial/healthcare sites |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    business_type   VARCHAR(50) NOT NULL DEFAULT 'residential',  -- residential, commercial, mixed, healthcare
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings: {
    --   "booking_lead_time_hours": 24,
    --   "geofence_radius_meters": 100,
    --   "auto_invoice": true,
    --   "review_platforms": ["google", "facebook"],
    --   "locale_options": ["en", "es"],
    --   "cims_tracking_enabled": false,
    --   "leed_tracking_enabled": false,
    --   "hipaa_mode": false
    -- }
    compliance_config JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "cims_version": "2018",
    --   "required_certifications": ["epa_safer_choice"],
    --   "sds_expiry_alert_days": 30
    -- }
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
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"notifications": {"sms": true, "email": true, "push": true}, "calendar_view": "week"}
    auth_providers   JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"provider": "google", "uid": "1234567890"}]
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_user_tenant ON "user"(tenant_id);
CREATE INDEX idx_user_email ON "user"(email);
```

## Clients & Properties

```sql
CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    -- Universal relational fields
    display_name    VARCHAR(255) NOT NULL,
    is_company      BOOLEAN NOT NULL DEFAULT false,
    email           VARCHAR(255),
    phone           VARCHAR(30),
    source          VARCHAR(50),
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    tags            TEXT[],
    -- Variable fields
    contact_info    JSONB NOT NULL DEFAULT '{}',
    -- Example (residential): {
    --   "first_name": "Jane", "last_name": "Smith",
    --   "secondary_phone": "555-0102",
    --   "preferred_contact_method": "sms"
    -- }
    -- Example (commercial): {
    --   "company_name": "Acme Corp",
    --   "primary_contact": {"name": "John Doe", "title": "Facilities Manager", "email": "john@acme.com"},
    --   "accounts_payable_contact": {"name": "Jane Finance", "email": "ap@acme.com"},
    --   "purchase_order_required": true
    -- }
    billing_address JSONB NOT NULL DEFAULT '{}',
    -- Example: {"line1": "123 Main St", "city": "Austin", "state": "TX", "postal_code": "78701", "country": "US"}
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Tenant-defined custom fields
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_client_tenant ON client(tenant_id);
CREATE INDEX idx_client_name ON client(tenant_id, display_name);
CREATE INDEX idx_client_tags ON client USING GIN (tags);

CREATE TABLE property (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    name            VARCHAR(255),
    property_type   VARCHAR(50) NOT NULL DEFAULT 'residential',
    -- Universal location fields (relational for geospatial queries)
    address_line1   VARCHAR(255) NOT NULL,
    city            VARCHAR(100) NOT NULL,
    state           VARCHAR(100),
    postal_code     VARCHAR(20),
    country         CHAR(2) NOT NULL DEFAULT 'US',
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    -- Variable metadata depending on property type
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example (residential): {
    --   "square_footage": 2200,
    --   "bedrooms": 3, "bathrooms": 2.5,
    --   "floors": 2, "has_pets": true, "pet_type": "dog",
    --   "alarm_code": "1234", "lockbox_code": "5678",
    --   "parking": "driveway"
    -- }
    -- Example (commercial): {
    --   "square_footage": 45000,
    --   "floors": 4, "zones": ["lobby", "offices_2f", "offices_3f", "restrooms"],
    --   "gln": "0614141000012",
    --   "building_manager": {"name": "Bob", "phone": "555-0199"},
    --   "access_hours": {"start": "18:00", "end": "06:00"},
    --   "security_clearance_required": true,
    --   "leed_certified": true, "leed_version": "v4.1"
    -- }
    -- Example (healthcare): {
    --   "square_footage": 80000,
    --   "gln": "0614141000029",
    --   "hipaa_site": true,
    --   "infection_control_level": "high",
    --   "restricted_areas": ["operating_rooms", "pharmacy", "nicu"],
    --   "required_certifications": ["bloodborne_pathogen", "hipaa_training"],
    --   "cleaning_frequency_override": {"operating_rooms": "after_each_procedure"}
    -- }
    compliance_requirements JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "leed_green_cleaning": true,
    --   "epa_safer_choice_only": true,
    --   "cims_reporting_required": true,
    --   "hipaa_compliant_logging": false
    -- }
    access_instructions TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_property_tenant ON property(tenant_id);
CREATE INDEX idx_property_client ON property(client_id);
CREATE INDEX idx_property_location ON property(latitude, longitude);
CREATE INDEX idx_property_type ON property(tenant_id, property_type);
CREATE INDEX idx_property_metadata ON property USING GIN (metadata jsonb_path_ops);
CREATE INDEX idx_property_compliance ON property USING GIN (compliance_requirements jsonb_path_ops);
```

## Workforce

```sql
CREATE TABLE cleaner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES "user"(id),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(30),
    employment_type VARCHAR(20) NOT NULL DEFAULT 'employee',
    hourly_rate     DECIMAL(10, 2),
    preferred_locale VARCHAR(10) NOT NULL DEFAULT 'en',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Variable cleaner data
    home_location   JSONB,
    -- Example: {"address": "456 Oak Ave", "city": "Austin", "state": "TX", "latitude": 30.2672, "longitude": -97.7431}
    skills          JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"name": "deep_clean", "proficiency": "expert", "certified_at": "2025-06-15"},
    --           {"name": "healthcare", "proficiency": "standard", "certified_at": "2026-01-10"}]
    availability    JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"day": 1, "start": "08:00", "end": "17:00", "available": true},
    --           {"day": 2, "start": "08:00", "end": "17:00", "available": true}]
    certifications  JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"type": "bloodborne_pathogen", "issued": "2025-11-01", "expires": "2026-11-01"},
    --           {"type": "hipaa_training", "issued": "2025-09-15", "expires": "2026-09-15"}]
    performance     JSONB NOT NULL DEFAULT '{}',
    -- Denormalized, periodically recalculated
    -- Example: {"total_jobs": 342, "avg_duration_min": 98, "avg_inspection_score": 4.3,
    --           "hours_this_week": 32.5, "retention_risk_score": 0.12}
    hire_date       DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cleaner_tenant ON cleaner(tenant_id);
CREATE INDEX idx_cleaner_active ON cleaner(tenant_id, is_active);
CREATE INDEX idx_cleaner_skills ON cleaner USING GIN (skills jsonb_path_ops);

CREATE TABLE team (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    lead_cleaner_id UUID REFERENCES cleaner(id),
    member_ids      UUID[] NOT NULL DEFAULT '{}',
    -- Array of cleaner IDs — simple for small teams
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_team_tenant ON team(tenant_id);
```

## Service Catalogue

```sql
CREATE TABLE service_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    default_duration_minutes INTEGER NOT NULL DEFAULT 120,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Flexible pricing rules
    pricing         JSONB NOT NULL DEFAULT '{}',
    -- Example (residential): {
    --   "model": "flat_plus_sqft",
    --   "base_price": 80.00,
    --   "price_per_sqft": 0.05,
    --   "bedroom_upcharge": 15.00,
    --   "bathroom_upcharge": 20.00,
    --   "pet_surcharge": 10.00
    -- }
    -- Example (commercial): {
    --   "model": "hourly",
    --   "hourly_rate": 35.00,
    --   "minimum_hours": 2,
    --   "weekend_multiplier": 1.5
    -- }
    -- Example (dynamic): {
    --   "model": "dynamic",
    --   "base_price": 100.00,
    --   "frequency_discount": {"weekly": 0.15, "biweekly": 0.10, "monthly": 0.0},
    --   "market_rate_adjustment": true
    -- }
    addons          JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"name": "Inside Fridge", "price": 25.00, "duration_min": 20},
    --           {"name": "Inside Oven", "price": 20.00, "duration_min": 15},
    --           {"name": "Laundry (2 loads)", "price": 30.00, "duration_min": 40}]
    required_skills JSONB NOT NULL DEFAULT '[]',
    -- Example: ["deep_clean", "carpet"]
    default_checklist JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"section": "Kitchen", "items": ["Countertops", "Sink", "Stovetop", "Floor"]},
    --           {"section": "Bathroom", "items": ["Toilet", "Shower/tub", "Mirror", "Floor"]}]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);
```

## Jobs & Scheduling

```sql
CREATE TABLE schedule_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    property_id     UUID NOT NULL REFERENCES property(id),
    service_type_id UUID NOT NULL REFERENCES service_type(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- RFC 5545-compatible recurrence stored as JSONB
    recurrence      JSONB NOT NULL,
    -- Example: {"freq": "WEEKLY", "interval": 1, "byday": ["TH"], "dtstart": "2026-05-01",
    --           "preferred_time": "09:00", "preferred_end": "11:00", "until": null}
    selected_addons JSONB NOT NULL DEFAULT '[]',
    -- Example: ["Inside Fridge", "Laundry (2 loads)"]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_schedule_rule_tenant ON schedule_rule(tenant_id);

CREATE TABLE job (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    job_number      SERIAL,
    schedule_rule_id UUID REFERENCES schedule_rule(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    property_id     UUID NOT NULL REFERENCES property(id),
    service_type_id UUID NOT NULL REFERENCES service_type(id),
    -- Universal relational fields (queried constantly)
    status          VARCHAR(30) NOT NULL DEFAULT 'scheduled',
    priority        VARCHAR(10) NOT NULL DEFAULT 'normal',
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end   TIMESTAMPTZ NOT NULL,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    quoted_price    DECIMAL(10, 2),
    final_price     DECIMAL(10, 2),
    -- Variable job data
    assignments     JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"cleaner_id": "uuid", "name": "Maria Garcia", "role": "lead", "status": "accepted"},
    --           {"cleaner_id": "uuid", "name": "Carlos Lopez", "role": "helper", "status": "pending"}]
    selected_addons JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"name": "Inside Fridge", "price": 25.00, "completed": true}]
    checklist       JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"section": "Kitchen", "items": [
    --   {"description": "Countertops", "completed": true, "completed_by": "uuid", "completed_at": "..."},
    --   {"description": "Sink", "completed": false}
    -- ]}]
    clock_events    JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"cleaner_id": "uuid", "type": "clock_in", "at": "2026-05-22T09:03:00Z",
    --            "lat": 30.2672, "lng": -97.7431, "within_geofence": true}]
    photos          JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"storage_key": "s3://...", "type": "before", "caption": "Kitchen before",
    --            "taken_at": "2026-05-22T09:05:00Z", "lat": 30.2672, "lng": -97.7431}]
    product_usage   JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"product_id": "uuid", "product_name": "EcoClean Multi", "quantity": 50, "unit": "ml"}]
    instructions    TEXT,
    internal_notes  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_job_tenant ON job(tenant_id);
CREATE INDEX idx_job_scheduled ON job(tenant_id, scheduled_start);
CREATE INDEX idx_job_status ON job(tenant_id, status);
CREATE INDEX idx_job_client ON job(client_id);
CREATE INDEX idx_job_property ON job(property_id);
CREATE INDEX idx_job_assignments ON job USING GIN (assignments jsonb_path_ops);
```

## Quality Inspection

```sql
CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    job_id          UUID REFERENCES job(id),
    property_id     UUID NOT NULL REFERENCES property(id),
    inspector_id    UUID NOT NULL REFERENCES "user"(id),
    template_name   VARCHAR(255) NOT NULL,
    template_version INTEGER NOT NULL DEFAULT 1,
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress',
    overall_score   DECIMAL(5, 2),
    -- The full inspection data lives in JSONB
    sections        JSONB NOT NULL DEFAULT '[]',
    -- Example: [{
    --   "name": "Kitchen", "weight": 1.0, "score": 4.2,
    --   "items": [
    --     {"description": "Countertops wiped and sanitised", "rating": "meets",
    --      "score": 4, "notes": "", "photos": ["s3://..."]},
    --     {"description": "Floor mopped — no streaks", "rating": "exceeds",
    --      "score": 5, "notes": "Excellent work", "photos": []}
    --   ]
    -- }, {
    --   "name": "Restrooms", "weight": 1.5, "score": 3.8,
    --   "items": [
    --     {"description": "Toilet sanitised inside and out", "rating": "doesnt_meet",
    --      "score": 2, "notes": "Residue around base", "photos": ["s3://...", "s3://..."]}
    --   ]
    -- }]
    location        JSONB,
    -- Example: {"latitude": 30.2672, "longitude": -97.7431, "accuracy_meters": 5}
    inspected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    submitted_at    TIMESTAMPTZ,
    report_sent_at  TIMESTAMPTZ,
    report_recipients TEXT[],
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_tenant ON inspection(tenant_id);
CREATE INDEX idx_inspection_property ON inspection(property_id);
CREATE INDEX idx_inspection_job ON inspection(job_id);
CREATE INDEX idx_inspection_sections ON inspection USING GIN (sections jsonb_path_ops);
```

## Billing & Payments

```sql
CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    invoice_number  VARCHAR(50) NOT NULL,
    client_id       UUID NOT NULL REFERENCES client(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    subtotal        DECIMAL(10, 2) NOT NULL DEFAULT 0,
    tax_rate        DECIMAL(5, 4) DEFAULT 0,
    tax_amount      DECIMAL(10, 2) DEFAULT 0,
    total           DECIMAL(10, 2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    due_date        DATE,
    line_items      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"job_id": "uuid", "description": "Standard Clean — 123 Main St (May 22)",
    --            "quantity": 1, "unit_price": 150.00, "total": 150.00},
    --           {"description": "Inside Fridge add-on", "quantity": 1, "unit_price": 25.00, "total": 25.00}]
    payments        JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"payment_id": "uuid", "amount": 175.00, "method": "card",
    --            "stripe_id": "pi_...", "paid_at": "2026-05-22T15:00:00Z"}]
    notes           TEXT,
    sent_at         TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, invoice_number)
);

CREATE INDEX idx_invoice_tenant ON invoice(tenant_id);
CREATE INDEX idx_invoice_client ON invoice(client_id);
CREATE INDEX idx_invoice_status ON invoice(tenant_id, status);

CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    invoice_id      UUID REFERENCES invoice(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    amount          DECIMAL(10, 2) NOT NULL,
    payment_method  VARCHAR(30) NOT NULL,
    stripe_payment_id VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    paid_at         TIMESTAMPTZ,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example: {"card_last4": "4242", "card_brand": "visa", "receipt_url": "https://..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_invoice ON payment(invoice_id);

CREATE TABLE quote (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    quote_number    VARCHAR(50) NOT NULL,
    client_id       UUID NOT NULL REFERENCES client(id),
    property_id     UUID NOT NULL REFERENCES property(id),
    service_type_id UUID NOT NULL REFERENCES service_type(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    total           DECIMAL(10, 2) NOT NULL DEFAULT 0,
    line_items      JSONB NOT NULL DEFAULT '[]',
    valid_until     DATE,
    message         TEXT,
    sent_at         TIMESTAMPTZ,
    responded_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, quote_number)
);
```

## Chemical & Product Compliance

```sql
CREATE TABLE chemical_product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    manufacturer    VARCHAR(255),
    -- Relational fields for common queries
    is_epa_safer_choice BOOLEAN NOT NULL DEFAULT false,
    is_leed_compliant   BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Variable product data
    identifiers     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"product_code": "EC-100", "cas_number": "7732-18-5", "upc": "012345678901"}
    hazard_info     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"hazard_class": "Corrosive", "ghs_pictograms": ["GHS05"],
    --           "signal_word": "Danger", "h_statements": ["H314"],
    --           "p_statements": ["P260", "P264", "P280"]}
    certifications  JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"type": "epa_safer_choice", "certificate": "SC-2025-1234", "valid_until": "2027-06-30"},
    --           {"type": "green_seal", "certificate": "GS-37-2024", "valid_until": "2026-12-31"}]
    sds_info        JSONB NOT NULL DEFAULT '{}',
    -- Example: {"storage_key": "s3://sds/ecocleaner-100.pdf", "revision_date": "2025-03-15",
    --           "language": "en", "pages": 8}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_chemical_tenant ON chemical_product(tenant_id);
CREATE INDEX idx_chemical_certifications ON chemical_product USING GIN (certifications jsonb_path_ops);
```

## Equipment

```sql
CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    equipment_type  VARCHAR(50) NOT NULL,
    serial_number   VARCHAR(100),
    status          VARCHAR(20) NOT NULL DEFAULT 'available',
    assigned_to     UUID REFERENCES cleaner(id),
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"purchase_date": "2025-01-15", "purchase_price": 899.99,
    --           "warranty_expiry": "2027-01-15", "manufacturer": "ProTeam",
    --           "model": "Super CoachVac", "maintenance_schedule": "quarterly",
    --           "last_maintenance": "2026-03-01", "next_maintenance": "2026-06-01"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_tenant ON equipment(tenant_id);
```

## Routes

```sql
CREATE TABLE route (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    cleaner_id      UUID NOT NULL REFERENCES cleaner(id),
    route_date      DATE NOT NULL,
    is_optimised    BOOLEAN NOT NULL DEFAULT false,
    total_distance_km DECIMAL(8, 2),
    total_duration_minutes INTEGER,
    stops           JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"stop_order": 1, "job_id": "uuid", "address": "123 Main St",
    --    "latitude": 30.2672, "longitude": -97.7431,
    --    "estimated_arrival": "2026-05-22T09:00:00Z",
    --    "estimated_departure": "2026-05-22T11:00:00Z",
    --    "travel_distance_km": 5.2, "travel_duration_min": 12},
    --   {"stop_order": 2, "job_id": "uuid", "address": "456 Oak Ave", ...}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_route_tenant ON route(tenant_id, route_date);
CREATE INDEX idx_route_cleaner ON route(cleaner_id, route_date);
```

## Contracts (Commercial)

```sql
CREATE TABLE contract (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    contract_number VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    start_date      DATE NOT NULL,
    end_date        DATE,
    monthly_value   DECIMAL(10, 2),
    -- Variable contract terms
    terms           JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "billing_frequency": "monthly",
    --   "auto_renew": true,
    --   "renewal_notice_days": 30,
    --   "properties": ["uuid-1", "uuid-2"],
    --   "services": [
    --     {"service_type_id": "uuid", "frequency": "daily", "sessions_per_week": 5,
    --      "price_per_session": 150.00, "zones": ["lobby", "restrooms"]},
    --     {"service_type_id": "uuid", "frequency": "weekly", "sessions_per_week": 1,
    --      "price_per_session": 500.00, "zones": ["offices_2f", "offices_3f"]}
    --   ],
    --   "sla": {"max_response_time_hours": 4, "inspection_frequency": "weekly",
    --           "minimum_score": 4.0},
    --   "compliance_requirements": ["cims", "epa_safer_choice", "leed_v41"],
    --   "insurance_minimum": 2000000
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, contract_number)
);

CREATE INDEX idx_contract_tenant ON contract(tenant_id);
CREATE INDEX idx_contract_client ON contract(client_id);
CREATE INDEX idx_contract_terms ON contract USING GIN (terms jsonb_path_ops);
```

## Communications & Audit Log

```sql
CREATE TABLE communication_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    channel         VARCHAR(20) NOT NULL,
    direction       VARCHAR(10) NOT NULL,
    recipient_type  VARCHAR(20) NOT NULL,
    recipient_id    UUID NOT NULL,
    job_id          UUID REFERENCES job(id),
    subject         VARCHAR(255),
    body            TEXT,
    external_id     VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'sent',
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example: {"template": "appointment_reminder", "twilio_sid": "SM...", "segments": 1}
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comm_log_tenant ON communication_log(tenant_id, created_at);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(30) NOT NULL,  -- create, update, delete, archive
    changes         JSONB,
    -- Example: {"status": {"old": "scheduled", "new": "completed"}, "final_price": {"old": null, "new": 150.00}}
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
```

---

## Example: Querying JSONB Fields

```sql
-- Find all properties requiring EPA Safer Choice products
SELECT id, name, address_line1, city
FROM property
WHERE tenant_id = 'tenant-uuid'
  AND compliance_requirements @> '{"epa_safer_choice_only": true}';

-- Find all cleaners with the "healthcare" skill at expert level
SELECT id, first_name, last_name
FROM cleaner
WHERE tenant_id = 'tenant-uuid'
  AND skills @> '[{"name": "healthcare", "proficiency": "expert"}]';

-- Find all chemical products with expired certifications
SELECT id, name, cert->>'type' AS certification, cert->>'valid_until' AS expires
FROM chemical_product,
     jsonb_array_elements(certifications) AS cert
WHERE tenant_id = 'tenant-uuid'
  AND (cert->>'valid_until')::date < CURRENT_DATE;

-- Find all jobs where a specific cleaner is assigned
SELECT id, job_number, status, scheduled_start
FROM job
WHERE tenant_id = 'tenant-uuid'
  AND assignments @> '[{"cleaner_id": "specific-cleaner-uuid"}]';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenant, user |
| Clients & Properties | 2 | client, property (with rich JSONB metadata) |
| Workforce | 2 | cleaner (skills/availability in JSONB), team |
| Service Catalogue | 1 | service_type (pricing/addons/checklist in JSONB) |
| Scheduling & Jobs | 2 | schedule_rule, job (assignments/checklist/photos in JSONB) |
| Quality Inspection | 1 | inspection (full section/item results in JSONB) |
| Billing | 3 | invoice (line_items in JSONB), payment, quote |
| Chemical Compliance | 1 | chemical_product (hazard/certifications in JSONB) |
| Equipment | 1 | equipment (details in JSONB) |
| Routes | 1 | route (stops in JSONB) |
| Contracts | 1 | contract (terms/services/SLA in JSONB) |
| Communications & Audit | 2 | communication_log, audit_log |
| **Total** | **~19** | Significantly fewer tables; complexity in JSONB structures |

---

## Key Design Decisions

1. **Relational columns for query-critical, universal fields; JSONB for variable, context-dependent data** — fields that appear in WHERE clauses, ORDER BY, and JOIN conditions (status, scheduled_start, tenant_id, client_id) are always relational. Fields that vary by property type, business segment, or jurisdiction live in JSONB.

2. **Only 19 tables vs ~46 in the normalized model** — the hybrid approach collapses junction tables (cleaner_skill, job_assignment, job_addon, etc.) and detail tables (invoice_line_item, route_stop, inspection_result) into JSONB arrays on their parent entities. This dramatically simplifies the schema but requires application-level validation.

3. **GIN indexes on all JSONB columns** — every JSONB column that will be queried has a `jsonb_path_ops` GIN index, enabling efficient containment queries (`@>`) without full table scans.

4. **JSON Schema definitions documented per column** — while PostgreSQL does not enforce JSON Schema natively, each JSONB column has a documented schema definition in the application code. Validation happens at the API layer before writes.

5. **Denormalized performance metrics on cleaner** — the `performance` JSONB on cleaner stores pre-calculated aggregates (total_jobs, avg_inspection_score, retention_risk_score) that would otherwise require expensive queries. A background job recalculates these periodically.

6. **Audit log as a simple append-only table** — rather than full event sourcing, the hybrid model uses a lightweight `audit_log` table that captures entity changes as JSONB diffs. This provides compliance-adequate audit trails without the architectural complexity of CQRS.

7. **Contract terms as JSONB** — commercial cleaning contracts have wildly variable terms (SLAs, zones, compliance requirements, insurance minimums). Modelling each variation as a relational column would be unmanageable; JSONB captures the full contract structure while the relational `status` and `monthly_value` fields support dashboard queries.

8. **Inspection sections/items embedded in the inspection row** — a single inspection record contains its entire result set as JSONB. This eliminates the need for separate `inspection_result` and `inspection_photo` tables. The trade-off is that cross-inspection item-level analytics requires JSONB extraction, but this is acceptable given that inspection-level queries (score, status, property) are far more common.

9. **RFC 5545 recurrence as JSONB** — the `recurrence` JSONB on `schedule_rule` mirrors iCalendar RRULE structure, making it straightforward to import/export to calendar applications while avoiding the rigidity of separate columns for each recurrence parameter.

10. **Progressive migration path** — if a JSONB field proves to be queried so frequently that GIN index performance is insufficient, it can be promoted to a relational column in a future migration. The hybrid model is designed to evolve based on actual query patterns.
