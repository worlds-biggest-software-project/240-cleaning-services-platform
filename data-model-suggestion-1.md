# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Cleaning Services Platform · Created: 2026-05-22

## Philosophy

This model follows a fully normalized relational design where every domain concept occupies its own table, linked by foreign keys and junction tables. The schema enforces data integrity at the database level through constraints, unique indexes, and referential integrity rules. Every cleaning job, inspection result, chemical product, and compliance obligation is a first-class entity with its own lifecycle.

The approach draws from the field service data models used by Salesforce Field Service (Work Order / Service Appointment / Service Resource) and Microsoft Dynamics 365 Field Service (msdyn_workorder / msdyn_workorderservice / msdyn_workorderservicetask), adapted for the cleaning-specific domain. Reference data tables (jurisdictions, chemical hazard classes, CIMS compliance elements) are separated from transactional data, enabling clean reporting and standards alignment.

This is the most predictable and widely understood architectural pattern. It works well with standard ORM frameworks, produces straightforward SQL queries, and makes schema evolution explicit through migrations. The trade-off is a higher table count and more JOIN operations, but modern PostgreSQL handles this efficiently for the expected data volumes of a cleaning platform.

**Best for:** Teams that prioritize data integrity, complex cross-entity reporting, and regulatory compliance in a well-understood relational paradigm.

**Trade-offs:**
- (+) Strong referential integrity — the database enforces business rules
- (+) Straightforward reporting with standard SQL JOINs
- (+) Well-understood by most developers; excellent ORM support
- (+) Clear migration path as the schema evolves
- (-) High table count (~55-65 tables) increases schema complexity
- (-) Many-to-many relationships require junction tables, adding query complexity
- (-) Schema changes require migrations; less flexible for jurisdiction-specific variation
- (-) Recursive or hierarchical queries (location trees, organizational hierarchies) require CTEs

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISSA CIMS | Dedicated `cims_section`, `cims_element`, and `cims_compliance_record` tables model all five CIMS sections with mandatory/recommended element tracking |
| OSHA HazCom (29 CFR 1910.1200) | `chemical_product` and `safety_data_sheet` tables track SDS documents, CAS numbers, hazard classifications, and worker exposure records |
| EPA Safer Choice | `product_certification` table links chemical products to EPA Safer Choice, LEED, and other certifications |
| LEED v4.1 EQp3 | `property` table carries a `leed_certified` flag; cleaning product usage is tracked against green product requirements via `job_product_usage` |
| ISO 3166 | `jurisdiction` reference table uses ISO 3166-1 alpha-2 country codes and ISO 3166-2 subdivision codes |
| RFC 5545 (iCalendar) | `schedule_rule` table stores RRULE-compatible recurrence patterns for recurring jobs |
| GS1 GLN | `property` table includes an optional `gln` (Global Location Number) field for commercial/healthcare facility identification |
| OAuth 2.0 / OIDC | `user` and `auth_provider` tables support federated authentication |

---

## Core Identity & Multi-Tenancy

```sql
-- Every row in every tenant-scoped table carries tenant_id for RLS
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
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
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- owner, admin, manager, member, cleaner, client
    locale          VARCHAR(10) NOT NULL DEFAULT 'en',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_user_tenant ON "user"(tenant_id);
CREATE INDEX idx_user_email ON "user"(email);

CREATE TABLE auth_provider (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,  -- google, apple, microsoft, saml
    provider_uid    VARCHAR(255) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (provider, provider_uid)
);
```

## Client & Property Management

```sql
CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    is_company      BOOLEAN NOT NULL DEFAULT false,
    company_name    VARCHAR(255),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    email           VARCHAR(255),
    phone           VARCHAR(30),
    secondary_phone VARCHAR(30),
    billing_address_line1  VARCHAR(255),
    billing_address_line2  VARCHAR(255),
    billing_city           VARCHAR(100),
    billing_state          VARCHAR(100),
    billing_postal_code    VARCHAR(20),
    billing_country        CHAR(2) DEFAULT 'US',  -- ISO 3166-1 alpha-2
    notes           TEXT,
    source          VARCHAR(50),  -- website, referral, google, manual
    tags            TEXT[],
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_client_tenant ON client(tenant_id);
CREATE INDEX idx_client_name ON client(tenant_id, last_name, first_name);

CREATE TABLE property (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    name            VARCHAR(255),  -- "Main Office", "Home", "Building A"
    property_type   VARCHAR(50) NOT NULL DEFAULT 'residential',  -- residential, commercial, healthcare, hospitality
    address_line1   VARCHAR(255) NOT NULL,
    address_line2   VARCHAR(255),
    city            VARCHAR(100) NOT NULL,
    state           VARCHAR(100),
    postal_code     VARCHAR(20),
    country         CHAR(2) NOT NULL DEFAULT 'US',  -- ISO 3166-1 alpha-2
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    gln             VARCHAR(13),  -- GS1 Global Location Number for commercial sites
    square_footage  INTEGER,
    bedroom_count   SMALLINT,
    bathroom_count  SMALLINT,
    floor_count     SMALLINT,
    leed_certified  BOOLEAN NOT NULL DEFAULT false,
    hipaa_site      BOOLEAN NOT NULL DEFAULT false,  -- healthcare facility flag
    access_instructions TEXT,
    notes           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_property_tenant ON property(tenant_id);
CREATE INDEX idx_property_client ON property(client_id);
CREATE INDEX idx_property_location ON property(latitude, longitude);
```

## Workforce Management

```sql
CREATE TABLE cleaner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES "user"(id),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(30),
    home_address_line1  VARCHAR(255),
    home_city       VARCHAR(100),
    home_state      VARCHAR(100),
    home_postal_code VARCHAR(20),
    home_latitude   DECIMAL(10, 7),
    home_longitude  DECIMAL(10, 7),
    hourly_rate     DECIMAL(10, 2),
    employment_type VARCHAR(20) NOT NULL DEFAULT 'employee',  -- employee, contractor
    preferred_locale VARCHAR(10) NOT NULL DEFAULT 'en',
    hire_date       DATE,
    max_hours_per_week SMALLINT DEFAULT 40,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cleaner_tenant ON cleaner(tenant_id);

CREATE TABLE skill (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,  -- deep_clean, carpet, window, healthcare, hazmat
    description     TEXT,
    UNIQUE (tenant_id, name)
);

CREATE TABLE cleaner_skill (
    cleaner_id      UUID NOT NULL REFERENCES cleaner(id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skill(id) ON DELETE CASCADE,
    proficiency     VARCHAR(20) DEFAULT 'standard',  -- trainee, standard, expert
    certified_at    DATE,
    PRIMARY KEY (cleaner_id, skill_id)
);

CREATE TABLE cleaner_availability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cleaner_id      UUID NOT NULL REFERENCES cleaner(id) ON DELETE CASCADE,
    day_of_week     SMALLINT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),  -- 0=Sunday
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    is_available    BOOLEAN NOT NULL DEFAULT true
);

CREATE INDEX idx_cleaner_avail ON cleaner_availability(cleaner_id, day_of_week);

CREATE TABLE team (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    lead_cleaner_id UUID REFERENCES cleaner(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_member (
    team_id         UUID NOT NULL REFERENCES team(id) ON DELETE CASCADE,
    cleaner_id      UUID NOT NULL REFERENCES cleaner(id) ON DELETE CASCADE,
    role            VARCHAR(20) NOT NULL DEFAULT 'member',  -- lead, member
    PRIMARY KEY (team_id, cleaner_id)
);
```

## Service Types & Pricing

```sql
CREATE TABLE service_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,  -- standard_clean, deep_clean, move_in_out, post_construction
    description     TEXT,
    default_duration_minutes INTEGER NOT NULL DEFAULT 120,
    base_price      DECIMAL(10, 2),
    price_per_sqft  DECIMAL(6, 4),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE service_addon (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,  -- inside_fridge, inside_oven, laundry, window_interior
    price           DECIMAL(10, 2) NOT NULL,
    duration_minutes INTEGER NOT NULL DEFAULT 30,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (tenant_id, name)
);

CREATE TABLE service_type_skill_requirement (
    service_type_id UUID NOT NULL REFERENCES service_type(id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skill(id) ON DELETE CASCADE,
    is_required     BOOLEAN NOT NULL DEFAULT true,  -- required vs preferred
    PRIMARY KEY (service_type_id, skill_id)
);
```

## Scheduling & Jobs

```sql
CREATE TABLE schedule_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    property_id     UUID NOT NULL REFERENCES property(id),
    service_type_id UUID NOT NULL REFERENCES service_type(id),
    -- RFC 5545 RRULE-compatible recurrence
    frequency       VARCHAR(20) NOT NULL,  -- once, daily, weekly, biweekly, monthly
    interval_value  SMALLINT NOT NULL DEFAULT 1,
    day_of_week     SMALLINT,  -- 0-6
    preferred_start_time TIME,
    preferred_end_time   TIME,
    starts_on       DATE NOT NULL,
    ends_on         DATE,  -- NULL = indefinite
    is_active       BOOLEAN NOT NULL DEFAULT true,
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
    status          VARCHAR(30) NOT NULL DEFAULT 'scheduled',
                    -- scheduled, dispatched, in_progress, completed, cancelled, no_show
    priority        VARCHAR(10) NOT NULL DEFAULT 'normal',  -- low, normal, high, urgent
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end   TIMESTAMPTZ NOT NULL,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    estimated_duration_minutes INTEGER,
    instructions    TEXT,
    internal_notes  TEXT,
    quoted_price    DECIMAL(10, 2),
    final_price     DECIMAL(10, 2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_job_tenant ON job(tenant_id);
CREATE INDEX idx_job_client ON job(client_id);
CREATE INDEX idx_job_property ON job(property_id);
CREATE INDEX idx_job_scheduled ON job(tenant_id, scheduled_start);
CREATE INDEX idx_job_status ON job(tenant_id, status);

CREATE TABLE job_assignment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id          UUID NOT NULL REFERENCES job(id) ON DELETE CASCADE,
    cleaner_id      UUID NOT NULL REFERENCES cleaner(id),
    team_id         UUID REFERENCES team(id),
    role            VARCHAR(20) NOT NULL DEFAULT 'assigned',  -- assigned, lead, helper
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',   -- pending, accepted, declined, completed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (job_id, cleaner_id)
);

CREATE INDEX idx_job_assignment_cleaner ON job_assignment(cleaner_id);
CREATE INDEX idx_job_assignment_job ON job_assignment(job_id);

CREATE TABLE job_addon (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id          UUID NOT NULL REFERENCES job(id) ON DELETE CASCADE,
    service_addon_id UUID NOT NULL REFERENCES service_addon(id),
    price           DECIMAL(10, 2) NOT NULL,
    completed       BOOLEAN NOT NULL DEFAULT false
);

CREATE TABLE job_checklist_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id          UUID NOT NULL REFERENCES job(id) ON DELETE CASCADE,
    description     VARCHAR(500) NOT NULL,
    sort_order      SMALLINT NOT NULL DEFAULT 0,
    is_completed    BOOLEAN NOT NULL DEFAULT false,
    completed_by    UUID REFERENCES cleaner(id),
    completed_at    TIMESTAMPTZ,
    notes           TEXT
);

CREATE INDEX idx_job_checklist ON job_checklist_item(job_id);
```

## GPS Tracking & Clock-In/Out

```sql
CREATE TABLE clock_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    cleaner_id      UUID NOT NULL REFERENCES cleaner(id),
    job_id          UUID REFERENCES job(id),
    event_type      VARCHAR(20) NOT NULL,  -- clock_in, clock_out, break_start, break_end
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    accuracy_meters DECIMAL(6, 1),
    is_within_geofence BOOLEAN,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    device_info     VARCHAR(255)
);

CREATE INDEX idx_clock_event_cleaner ON clock_event(cleaner_id, recorded_at);
CREATE INDEX idx_clock_event_job ON clock_event(job_id);

CREATE TABLE gps_breadcrumb (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cleaner_id      UUID NOT NULL REFERENCES cleaner(id),
    latitude        DECIMAL(10, 7) NOT NULL,
    longitude       DECIMAL(10, 7) NOT NULL,
    accuracy_meters DECIMAL(6, 1),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partitioned by month for performance
CREATE INDEX idx_gps_breadcrumb_cleaner ON gps_breadcrumb(cleaner_id, recorded_at);
```

## Quality Inspection

```sql
CREATE TABLE inspection_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    property_type   VARCHAR(50),  -- NULL = all types; residential, commercial, healthcare
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE inspection_template_section (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES inspection_template(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,  -- "Kitchen", "Bathroom", "Common Areas"
    sort_order      SMALLINT NOT NULL DEFAULT 0,
    weight          DECIMAL(5, 2) NOT NULL DEFAULT 1.0  -- scoring weight
);

CREATE TABLE inspection_template_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id      UUID NOT NULL REFERENCES inspection_template_section(id) ON DELETE CASCADE,
    description     VARCHAR(500) NOT NULL,  -- "Countertops wiped and sanitised"
    rating_type     VARCHAR(20) NOT NULL DEFAULT 'pass_fail',  -- pass_fail, scale_1_5, exceeds_meets_doesnt
    is_required     BOOLEAN NOT NULL DEFAULT true,
    sort_order      SMALLINT NOT NULL DEFAULT 0
);

CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    job_id          UUID REFERENCES job(id),
    property_id     UUID NOT NULL REFERENCES property(id),
    template_id     UUID NOT NULL REFERENCES inspection_template(id),
    inspector_id    UUID NOT NULL REFERENCES "user"(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress',  -- in_progress, completed, submitted
    overall_score   DECIMAL(5, 2),  -- calculated from item scores
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    notes           TEXT,
    inspected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    submitted_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_tenant ON inspection(tenant_id);
CREATE INDEX idx_inspection_property ON inspection(property_id);
CREATE INDEX idx_inspection_job ON inspection(job_id);

CREATE TABLE inspection_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspection(id) ON DELETE CASCADE,
    template_item_id UUID NOT NULL REFERENCES inspection_template_item(id),
    rating          VARCHAR(20),  -- pass, fail, 1-5, exceeds, meets, doesnt_meet
    numeric_score   DECIMAL(5, 2),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_result ON inspection_result(inspection_id);

CREATE TABLE inspection_photo (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspection(id) ON DELETE CASCADE,
    result_id       UUID REFERENCES inspection_result(id),
    storage_key     VARCHAR(500) NOT NULL,  -- S3/cloud storage path
    caption         VARCHAR(255),
    photo_type      VARCHAR(20) NOT NULL DEFAULT 'evidence',  -- before, after, evidence, issue
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    taken_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Billing & Invoicing

```sql
CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    invoice_number  VARCHAR(50) NOT NULL,
    client_id       UUID NOT NULL REFERENCES client(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, sent, viewed, paid, overdue, void
    subtotal        DECIMAL(10, 2) NOT NULL DEFAULT 0,
    tax_rate        DECIMAL(5, 4) DEFAULT 0,
    tax_amount      DECIMAL(10, 2) DEFAULT 0,
    total           DECIMAL(10, 2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    due_date        DATE,
    sent_at         TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, invoice_number)
);

CREATE INDEX idx_invoice_tenant ON invoice(tenant_id);
CREATE INDEX idx_invoice_client ON invoice(client_id);
CREATE INDEX idx_invoice_status ON invoice(tenant_id, status);

CREATE TABLE invoice_line_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoice(id) ON DELETE CASCADE,
    job_id          UUID REFERENCES job(id),
    description     VARCHAR(500) NOT NULL,
    quantity        DECIMAL(10, 2) NOT NULL DEFAULT 1,
    unit_price      DECIMAL(10, 2) NOT NULL,
    total           DECIMAL(10, 2) NOT NULL,
    sort_order      SMALLINT NOT NULL DEFAULT 0
);

CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    invoice_id      UUID REFERENCES invoice(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    amount          DECIMAL(10, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    payment_method  VARCHAR(30) NOT NULL,  -- card, ach, cash, check
    stripe_payment_id VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, succeeded, failed, refunded
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_invoice ON payment(invoice_id);
CREATE INDEX idx_payment_client ON payment(client_id);

CREATE TABLE client_payment_method (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES client(id) ON DELETE CASCADE,
    stripe_pm_id    VARCHAR(255) NOT NULL,
    card_last4      CHAR(4),
    card_brand      VARCHAR(20),
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Quoting & Proposals

```sql
CREATE TABLE quote (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    quote_number    VARCHAR(50) NOT NULL,
    client_id       UUID NOT NULL REFERENCES client(id),
    property_id     UUID NOT NULL REFERENCES property(id),
    service_type_id UUID NOT NULL REFERENCES service_type(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, sent, viewed, accepted, declined, expired
    subtotal        DECIMAL(10, 2) NOT NULL DEFAULT 0,
    total           DECIMAL(10, 2) NOT NULL DEFAULT 0,
    message         TEXT,
    valid_until     DATE,
    sent_at         TIMESTAMPTZ,
    responded_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, quote_number)
);

CREATE TABLE quote_line_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_id        UUID NOT NULL REFERENCES quote(id) ON DELETE CASCADE,
    description     VARCHAR(500) NOT NULL,
    quantity        DECIMAL(10, 2) NOT NULL DEFAULT 1,
    unit_price      DECIMAL(10, 2) NOT NULL,
    is_optional     BOOLEAN NOT NULL DEFAULT false,
    sort_order      SMALLINT NOT NULL DEFAULT 0
);
```

## Chemical & Product Compliance

```sql
CREATE TABLE chemical_product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    manufacturer    VARCHAR(255),
    product_code    VARCHAR(100),
    cas_number      VARCHAR(20),  -- Chemical Abstracts Service registry number
    hazard_class    VARCHAR(50),  -- OSHA HazCom classification
    is_epa_safer_choice BOOLEAN NOT NULL DEFAULT false,
    is_leed_compliant   BOOLEAN NOT NULL DEFAULT false,
    sds_storage_key     VARCHAR(500),  -- path to uploaded SDS PDF
    sds_revision_date   DATE,
    notes           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE product_certification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES chemical_product(id) ON DELETE CASCADE,
    certification   VARCHAR(100) NOT NULL,  -- epa_safer_choice, leed_v41, cims_gb, green_seal, ecologo
    certificate_number VARCHAR(100),
    valid_from      DATE,
    valid_until     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE job_product_usage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id          UUID NOT NULL REFERENCES job(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES chemical_product(id),
    quantity_used   DECIMAL(10, 3),
    unit            VARCHAR(20),  -- ml, oz, grams
    applied_by      UUID REFERENCES cleaner(id),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_job_product_usage ON job_product_usage(job_id);
```

## Equipment & Asset Tracking

```sql
CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    equipment_type  VARCHAR(50) NOT NULL,  -- vacuum, floor_machine, carpet_extractor, pressure_washer
    serial_number   VARCHAR(100),
    purchase_date   DATE,
    purchase_price  DECIMAL(10, 2),
    warranty_expiry DATE,
    assigned_to     UUID REFERENCES cleaner(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'available',  -- available, in_use, maintenance, retired
    last_maintenance_date DATE,
    next_maintenance_date DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_tenant ON equipment(tenant_id);
```

## CIMS Compliance Tracking

```sql
CREATE TABLE cims_section (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(10) NOT NULL UNIQUE,  -- QS, SF, HR, HRS, OD
    name            VARCHAR(100) NOT NULL,
    -- QS=Quality Systems, SF=Service Delivery, HR=Human Resources,
    -- HRS=Health/Safety/Environment, OD=Management Commitment
    description     TEXT
);

CREATE TABLE cims_element (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    section_id      UUID NOT NULL REFERENCES cims_section(id),
    code            VARCHAR(20) NOT NULL UNIQUE,  -- e.g. QS-1, QS-2, HR-3
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    is_mandatory    BOOLEAN NOT NULL DEFAULT false,
    sort_order      SMALLINT NOT NULL DEFAULT 0
);

CREATE TABLE cims_compliance_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    element_id      UUID NOT NULL REFERENCES cims_element(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'not_started',  -- not_started, in_progress, compliant, non_compliant
    evidence_notes  TEXT,
    evidence_storage_key VARCHAR(500),
    reviewed_by     UUID REFERENCES "user"(id),
    reviewed_at     TIMESTAMPTZ,
    valid_until     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, element_id)
);

CREATE INDEX idx_cims_compliance_tenant ON cims_compliance_record(tenant_id);
```

## Communications & Notifications

```sql
CREATE TABLE communication_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    channel         VARCHAR(20) NOT NULL,  -- sms, email, push, in_app
    direction       VARCHAR(10) NOT NULL,  -- outbound, inbound
    recipient_type  VARCHAR(20) NOT NULL,  -- client, cleaner
    recipient_id    UUID NOT NULL,
    job_id          UUID REFERENCES job(id),
    subject         VARCHAR(255),
    body            TEXT,
    template_name   VARCHAR(100),
    external_id     VARCHAR(255),  -- Twilio SID, SendGrid message ID
    status          VARCHAR(20) NOT NULL DEFAULT 'sent',  -- queued, sent, delivered, failed, bounced
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comm_log_tenant ON communication_log(tenant_id, created_at);
CREATE INDEX idx_comm_log_recipient ON communication_log(recipient_type, recipient_id);

CREATE TABLE review_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    job_id          UUID NOT NULL REFERENCES job(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    platform        VARCHAR(20) NOT NULL,  -- google, facebook, yelp, internal
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, sent, completed, declined
    rating          SMALLINT,  -- 1-5
    sent_at         TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Route Optimisation

```sql
CREATE TABLE route (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    cleaner_id      UUID NOT NULL REFERENCES cleaner(id),
    route_date      DATE NOT NULL,
    optimised       BOOLEAN NOT NULL DEFAULT false,
    total_distance_km DECIMAL(8, 2),
    total_duration_minutes INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_route_tenant_date ON route(tenant_id, route_date);
CREATE INDEX idx_route_cleaner ON route(cleaner_id, route_date);

CREATE TABLE route_stop (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    route_id        UUID NOT NULL REFERENCES route(id) ON DELETE CASCADE,
    job_id          UUID NOT NULL REFERENCES job(id),
    stop_order      SMALLINT NOT NULL,
    estimated_arrival TIMESTAMPTZ,
    estimated_departure TIMESTAMPTZ,
    travel_distance_km DECIMAL(8, 2),
    travel_duration_minutes INTEGER
);

CREATE INDEX idx_route_stop ON route_stop(route_id, stop_order);
```

## Contracts (Commercial Cleaning)

```sql
CREATE TABLE contract (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    contract_number VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, active, suspended, expired, terminated
    start_date      DATE NOT NULL,
    end_date        DATE,
    monthly_value   DECIMAL(10, 2),
    billing_frequency VARCHAR(20) NOT NULL DEFAULT 'monthly',  -- weekly, biweekly, monthly
    auto_renew      BOOLEAN NOT NULL DEFAULT false,
    terms           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, contract_number)
);

CREATE TABLE contract_property (
    contract_id     UUID NOT NULL REFERENCES contract(id) ON DELETE CASCADE,
    property_id     UUID NOT NULL REFERENCES property(id),
    PRIMARY KEY (contract_id, property_id)
);

CREATE TABLE contract_service (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES contract(id) ON DELETE CASCADE,
    service_type_id UUID NOT NULL REFERENCES service_type(id),
    frequency       VARCHAR(20) NOT NULL,  -- daily, weekly, biweekly, monthly
    sessions_per_period SMALLINT NOT NULL DEFAULT 1,
    price_per_session DECIMAL(10, 2)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | tenant, user, auth_provider |
| Client & Property | 2 | client, property |
| Workforce | 5 | cleaner, skill, cleaner_skill, cleaner_availability, team, team_member |
| Service Catalogue | 3 | service_type, service_addon, service_type_skill_requirement |
| Scheduling & Jobs | 5 | schedule_rule, job, job_assignment, job_addon, job_checklist_item |
| GPS & Time Tracking | 2 | clock_event, gps_breadcrumb |
| Quality Inspection | 6 | inspection_template, inspection_template_section, inspection_template_item, inspection, inspection_result, inspection_photo |
| Billing & Payments | 4 | invoice, invoice_line_item, payment, client_payment_method |
| Quoting | 2 | quote, quote_line_item |
| Chemical & Compliance | 3 | chemical_product, product_certification, job_product_usage |
| Equipment | 1 | equipment |
| CIMS Compliance | 3 | cims_section, cims_element, cims_compliance_record |
| Communications | 2 | communication_log, review_request |
| Route Optimisation | 2 | route, route_stop |
| Contracts | 3 | contract, contract_property, contract_service |
| **Total** | **~46** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe for multi-tenant environments and future data federation across regions.

2. **tenant_id on every transactional table** — supports row-level security (RLS) policies in PostgreSQL for data isolation without schema-per-tenant overhead.

3. **Separate cleaner and user tables** — a cleaner may not have a user account (especially subcontractors or low-tech staff), while users may not be cleaners (admins, managers, clients).

4. **Schedule rules separated from jobs** — the `schedule_rule` table defines the recurrence pattern (aligned with RFC 5545 RRULE semantics), while individual `job` rows represent each materialised occurrence. This avoids storing infinite future jobs while preserving the recurrence definition.

5. **Inspection templates versioned separately** — templates can evolve (new checklist items added) without altering historical inspection records. The `version` field on the template and the foreign key from `inspection_result` to `inspection_template_item` preserve point-in-time accuracy.

6. **Chemical compliance as first-class entities** — OSHA SDS tracking, EPA Safer Choice, and LEED compliance are modelled as dedicated tables rather than metadata flags, allowing audit-grade compliance reporting.

7. **GPS breadcrumbs partitioned by time** — the `gps_breadcrumb` table will accumulate high volumes; time-based partitioning is recommended for production deployment.

8. **Contracts for commercial cleaning** — a dedicated contract model supports multi-property, multi-service commercial agreements that generate recurring jobs, distinct from the simpler residential scheduling model.

9. **Route optimisation as a first-class entity** — routes and stops are persisted so that optimisation results can be reviewed, compared, and replayed. The route model integrates with PostGIS/pgRouting for geospatial queries.

10. **Communication log for audit trails** — every SMS, email, and push notification is logged with external IDs (Twilio SID, SendGrid message ID) for deliverability tracking and dispute resolution.
