# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Cleaning Services Platform · Created: 2026-05-22

## Philosophy

This model combines a relational core for operational CRUD (scheduling, billing, invoicing) with a property graph layer for relationship-heavy queries. The cleaning services domain has several relationship patterns that are awkward in pure relational models but natural in graphs: cleaner-to-property affinity networks, client referral chains, equipment assignment histories, multi-property contract hierarchies, franchise/brand ownership structures, and conflict-of-interest detection (e.g., a cleaner who should not be assigned to a former employer's property).

The graph layer is implemented using PostgreSQL's `ltree` extension for hierarchies and a general-purpose `graph_node` / `graph_edge` adjacency model for arbitrary relationships. This avoids introducing a separate graph database (Neo4j, Amazon Neptune) while still enabling graph traversal queries. For teams that need richer graph capabilities, the `graph_edge` table can be replicated to a dedicated graph database as a read replica.

The relational tables handle the high-frequency operational workload: scheduling, clock-in/out, invoicing, and payments. The graph tables handle the analytical and intelligence workload: "which cleaners have the strongest affinity with this property?", "what is the referral chain that brought this client?", "which properties are connected through the same commercial contract hierarchy?", "are there any conflict-of-interest risks in today's assignments?"

**Best for:** Organisations with complex relationship networks — franchise operators, multi-brand cleaning companies, commercial contracts spanning property hierarchies, and platforms building AI features that rely on relationship intelligence (cleaner affinity, client referral value, conflict detection).

**Trade-offs:**
- (+) Natural representation of relationship-heavy queries (affinity, referral chains, hierarchies)
- (+) Enables AI features: cleaner-property affinity scoring, referral value analysis, conflict detection
- (+) PostgreSQL-native — no separate graph database required
- (+) Graph layer is additive — relational tables work independently if graph features are unused
- (+) `ltree` provides efficient hierarchical queries (franchise → region → branch → property)
- (-) Graph queries (recursive CTEs, ltree operations) are less familiar to most developers
- (-) Edge table grows rapidly with many relationships — requires pruning and indexing discipline
- (-) Two mental models required: relational for CRUD, graph for analytics
- (-) Maintaining graph edges in sync with relational state adds application complexity
- (-) Performance of deep graph traversals depends on edge count and index strategy

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISSA CIMS | CIMS compliance modelled as graph relationships between organisations, evidence documents, and certification elements — traversal queries identify compliance gaps across a franchise network |
| OSHA HazCom | Chemical product → SDS → worker exposure chain modelled as graph edges, enabling "which workers have been exposed to this chemical?" traversal queries |
| ISO 3166 | Geographic hierarchy (country → state → city → postal_code) modelled as an `ltree` path for efficient jurisdiction queries |
| GS1 GLN | Property nodes in the graph carry GLN identifiers; edges connect properties to contracts, zones, and parent facilities |
| RFC 5545 | Schedule recurrence stored in relational tables (not graph); graph edges connect schedules to the clients, properties, and cleaners involved |
| OAuth 2.0 / OIDC | User identity is relational; graph edges model user → role → tenant → permission relationships for complex RBAC |

---

## Graph Infrastructure

```sql
-- Enable ltree for hierarchical path queries
CREATE EXTENSION IF NOT EXISTS ltree;

-- General-purpose graph node table
-- Every domain entity that participates in relationships has a corresponding node
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    -- client, property, cleaner, team, contract, chemical_product, equipment,
    -- franchise, region, brand, zone, inspection_template
    entity_id       UUID NOT NULL,     -- FK to the relational entity
    label           VARCHAR(255),       -- human-readable label for display
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Type-specific attributes useful for graph queries
    -- Example (cleaner): {"skills": ["deep_clean", "healthcare"], "rating": 4.3}
    -- Example (property): {"type": "commercial", "sqft": 45000, "leed": true}
    hierarchy_path  LTREE,
    -- Example: "acme_corp.northeast.nyc.building_a"
    -- Used for franchise/org hierarchy queries
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, node_type, entity_id)
);

CREATE INDEX idx_graph_node_tenant ON graph_node(tenant_id);
CREATE INDEX idx_graph_node_type ON graph_node(node_type, entity_id);
CREATE INDEX idx_graph_node_hierarchy ON graph_node USING GIST (hierarchy_path);
CREATE INDEX idx_graph_node_properties ON graph_node USING GIN (properties jsonb_path_ops);

-- General-purpose graph edge table
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    -- Relationship types:
    -- ASSIGNED_TO:     cleaner → property (current assignment)
    -- HAS_AFFINITY:    cleaner → property (historical performance)
    -- REFERRED_BY:     client → client (referral chain)
    -- BELONGS_TO:      property → contract
    -- MANAGES:         user → team
    -- MEMBER_OF:       cleaner → team
    -- CHILD_OF:        property → property (building → zone)
    -- USES_PRODUCT:    job → chemical_product
    -- OWNS:            franchise → region, region → branch
    -- CERTIFIED_WITH:  chemical_product → certification
    -- CONFLICT:        cleaner → client (should not assign)
    -- INSPECTS:        user → property (inspector assignment)
    weight          DECIMAL(8, 4) DEFAULT 1.0,
    -- Semantic meaning depends on edge_type:
    -- HAS_AFFINITY: affinity score (0.0–1.0)
    -- REFERRED_BY: referral value ($)
    -- CONFLICT: severity (1=soft, 5=hard block)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example (HAS_AFFINITY): {"jobs_completed": 45, "avg_score": 4.7, "last_job": "2026-05-20"}
    -- Example (REFERRED_BY): {"referral_date": "2026-03-15", "referral_code": "JANE50"}
    -- Example (CONFLICT): {"reason": "former_employer", "reported_by": "uuid", "active": true}
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_until     TIMESTAMPTZ,  -- NULL = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edge_source ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_graph_edge_target ON graph_edge(target_node_id, edge_type);
CREATE INDEX idx_graph_edge_type ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_graph_edge_active ON graph_edge(tenant_id, edge_type)
    WHERE valid_until IS NULL;
CREATE INDEX idx_graph_edge_properties ON graph_edge USING GIN (properties jsonb_path_ops);
```

## Relational Core: Identity & Multi-Tenancy

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    business_type   VARCHAR(50) NOT NULL DEFAULT 'residential',
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
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_user_tenant ON "user"(tenant_id);
```

## Relational Core: Clients & Properties

```sql
CREATE TABLE client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    display_name    VARCHAR(255) NOT NULL,
    is_company      BOOLEAN NOT NULL DEFAULT false,
    company_name    VARCHAR(255),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    email           VARCHAR(255),
    phone           VARCHAR(30),
    billing_address JSONB NOT NULL DEFAULT '{}',
    source          VARCHAR(50),
    tags            TEXT[],
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_client_tenant ON client(tenant_id);

CREATE TABLE property (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    name            VARCHAR(255),
    property_type   VARCHAR(50) NOT NULL DEFAULT 'residential',
    address_line1   VARCHAR(255) NOT NULL,
    city            VARCHAR(100) NOT NULL,
    state           VARCHAR(100),
    postal_code     VARCHAR(20),
    country         CHAR(2) NOT NULL DEFAULT 'US',
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    gln             VARCHAR(13),
    metadata        JSONB NOT NULL DEFAULT '{}',
    compliance_requirements JSONB NOT NULL DEFAULT '{}',
    access_instructions TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_property_tenant ON property(tenant_id);
CREATE INDEX idx_property_client ON property(client_id);
CREATE INDEX idx_property_location ON property(latitude, longitude);
```

## Relational Core: Workforce

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
    home_latitude   DECIMAL(10, 7),
    home_longitude  DECIMAL(10, 7),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    skills          JSONB NOT NULL DEFAULT '[]',
    availability    JSONB NOT NULL DEFAULT '[]',
    hire_date       DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cleaner_tenant ON cleaner(tenant_id);
CREATE INDEX idx_cleaner_skills ON cleaner USING GIN (skills jsonb_path_ops);
```

## Relational Core: Jobs & Scheduling

```sql
CREATE TABLE service_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    default_duration_minutes INTEGER NOT NULL DEFAULT 120,
    pricing         JSONB NOT NULL DEFAULT '{}',
    addons          JSONB NOT NULL DEFAULT '[]',
    default_checklist JSONB NOT NULL DEFAULT '[]',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE schedule_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    property_id     UUID NOT NULL REFERENCES property(id),
    service_type_id UUID NOT NULL REFERENCES service_type(id),
    recurrence      JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE job (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    job_number      SERIAL,
    schedule_rule_id UUID REFERENCES schedule_rule(id),
    client_id       UUID NOT NULL REFERENCES client(id),
    property_id     UUID NOT NULL REFERENCES property(id),
    service_type_id UUID NOT NULL REFERENCES service_type(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'scheduled',
    priority        VARCHAR(10) NOT NULL DEFAULT 'normal',
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end   TIMESTAMPTZ NOT NULL,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    quoted_price    DECIMAL(10, 2),
    final_price     DECIMAL(10, 2),
    assignments     JSONB NOT NULL DEFAULT '[]',
    checklist       JSONB NOT NULL DEFAULT '[]',
    clock_events    JSONB NOT NULL DEFAULT '[]',
    photos          JSONB NOT NULL DEFAULT '[]',
    instructions    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_job_tenant ON job(tenant_id);
CREATE INDEX idx_job_scheduled ON job(tenant_id, scheduled_start);
CREATE INDEX idx_job_status ON job(tenant_id, status);
CREATE INDEX idx_job_client ON job(client_id);
CREATE INDEX idx_job_property ON job(property_id);
```

## Relational Core: Inspections, Billing, Compliance

```sql
CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    job_id          UUID REFERENCES job(id),
    property_id     UUID NOT NULL REFERENCES property(id),
    inspector_id    UUID NOT NULL REFERENCES "user"(id),
    template_name   VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress',
    overall_score   DECIMAL(5, 2),
    sections        JSONB NOT NULL DEFAULT '[]',
    inspected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_tenant ON inspection(tenant_id);
CREATE INDEX idx_inspection_property ON inspection(property_id);

CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    invoice_number  VARCHAR(50) NOT NULL,
    client_id       UUID NOT NULL REFERENCES client(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    total           DECIMAL(10, 2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    line_items      JSONB NOT NULL DEFAULT '[]',
    due_date        DATE,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, invoice_number)
);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE chemical_product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    manufacturer    VARCHAR(255),
    is_epa_safer_choice BOOLEAN NOT NULL DEFAULT false,
    is_leed_compliant   BOOLEAN NOT NULL DEFAULT false,
    hazard_info     JSONB NOT NULL DEFAULT '{}',
    certifications  JSONB NOT NULL DEFAULT '[]',
    sds_info        JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    terms           JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, contract_number)
);

CREATE TABLE route (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    cleaner_id      UUID NOT NULL REFERENCES cleaner(id),
    route_date      DATE NOT NULL,
    is_optimised    BOOLEAN NOT NULL DEFAULT false,
    total_distance_km DECIMAL(8, 2),
    stops           JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE communication_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    channel         VARCHAR(20) NOT NULL,
    direction       VARCHAR(10) NOT NULL,
    recipient_type  VARCHAR(20) NOT NULL,
    recipient_id    UUID NOT NULL,
    job_id          UUID REFERENCES job(id),
    body            TEXT,
    external_id     VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'sent',
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comm_log_tenant ON communication_log(tenant_id, created_at);

CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    equipment_type  VARCHAR(50) NOT NULL,
    serial_number   VARCHAR(100),
    status          VARCHAR(20) NOT NULL DEFAULT 'available',
    assigned_to     UUID REFERENCES cleaner(id),
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Graph-Powered Query Examples

### Cleaner-Property Affinity: "Best cleaner for this property"

```sql
-- When a cleaner completes a job at a property, update the affinity edge
-- Application code runs after each job.completed:

-- Upsert affinity edge
INSERT INTO graph_edge (tenant_id, source_node_id, target_node_id, edge_type, weight, properties)
SELECT
    e_cleaner.tenant_id,
    e_cleaner.id,
    e_property.id,
    'HAS_AFFINITY',
    -- Calculate affinity from job count and inspection scores
    LEAST(1.0, (COALESCE((existing.properties->>'jobs_completed')::int, 0) + 1) * 0.02
              + COALESCE(i.overall_score, 4.0) * 0.1),
    jsonb_build_object(
        'jobs_completed', COALESCE((existing.properties->>'jobs_completed')::int, 0) + 1,
        'avg_score', COALESCE(i.overall_score, 4.0),
        'last_job', now()
    )
FROM graph_node e_cleaner
JOIN graph_node e_property ON e_property.node_type = 'property' AND e_property.entity_id = 'property-uuid'
LEFT JOIN graph_edge existing ON existing.source_node_id = e_cleaner.id
    AND existing.target_node_id = e_property.id
    AND existing.edge_type = 'HAS_AFFINITY'
LEFT JOIN inspection i ON i.job_id = 'completed-job-uuid'
WHERE e_cleaner.node_type = 'cleaner' AND e_cleaner.entity_id = 'cleaner-uuid'
ON CONFLICT (id) DO UPDATE SET
    weight = EXCLUDED.weight,
    properties = EXCLUDED.properties;

-- Query: Find the best 5 cleaners for a given property
SELECT
    c.first_name, c.last_name,
    e.weight AS affinity_score,
    e.properties->>'jobs_completed' AS jobs_at_property,
    e.properties->>'avg_score' AS avg_score
FROM graph_edge e
JOIN graph_node source_node ON e.source_node_id = source_node.id
JOIN graph_node target_node ON e.target_node_id = target_node.id
JOIN cleaner c ON source_node.entity_id = c.id
WHERE target_node.entity_id = 'property-uuid'
  AND target_node.node_type = 'property'
  AND e.edge_type = 'HAS_AFFINITY'
  AND e.valid_until IS NULL
  AND c.is_active = true
ORDER BY e.weight DESC
LIMIT 5;
```

### Client Referral Chain: "Who referred this client, and what's the total value?"

```sql
-- Recursive CTE to walk the referral chain
WITH RECURSIVE referral_chain AS (
    -- Start from the target client
    SELECT
        target_node.entity_id AS client_id,
        source_node.entity_id AS referred_by_id,
        e.properties->>'referral_code' AS referral_code,
        1 AS depth,
        ARRAY[target_node.entity_id] AS visited
    FROM graph_edge e
    JOIN graph_node target_node ON e.target_node_id = target_node.id
    JOIN graph_node source_node ON e.source_node_id = source_node.id
    WHERE target_node.entity_id = 'client-uuid'
      AND e.edge_type = 'REFERRED_BY'
      AND e.valid_until IS NULL

    UNION ALL

    -- Walk up the referral tree
    SELECT
        rc.referred_by_id AS client_id,
        source_node.entity_id AS referred_by_id,
        e.properties->>'referral_code' AS referral_code,
        rc.depth + 1,
        rc.visited || rc.referred_by_id
    FROM referral_chain rc
    JOIN graph_node target_node ON target_node.entity_id = rc.referred_by_id
        AND target_node.node_type = 'client'
    JOIN graph_edge e ON e.target_node_id = target_node.id
        AND e.edge_type = 'REFERRED_BY'
        AND e.valid_until IS NULL
    JOIN graph_node source_node ON e.source_node_id = source_node.id
    WHERE rc.depth < 10
      AND NOT rc.referred_by_id = ANY(rc.visited)  -- prevent cycles
)
SELECT
    rc.depth,
    c.display_name,
    rc.referral_code,
    COALESCE(SUM(p.amount), 0) AS total_revenue
FROM referral_chain rc
JOIN client c ON rc.client_id = c.id
LEFT JOIN payment p ON p.client_id = rc.client_id AND p.status = 'succeeded'
GROUP BY rc.depth, c.display_name, rc.referral_code
ORDER BY rc.depth;
```

### Franchise Hierarchy: "All properties under the Northeast region"

```sql
-- Using ltree for hierarchy queries
-- Hierarchy path example: "acme_corp.northeast.nyc"

-- Find all properties under the "northeast" region
SELECT
    gn.label,
    gn.properties->>'type' AS property_type,
    p.address_line1, p.city, p.state
FROM graph_node gn
JOIN property p ON gn.entity_id = p.id
WHERE gn.node_type = 'property'
  AND gn.hierarchy_path <@ 'acme_corp.northeast';
  -- ltree @> descendant-or-self operator

-- Find the full hierarchy above a specific property
SELECT
    gn.label, gn.node_type, gn.hierarchy_path
FROM graph_node gn
WHERE gn.hierarchy_path @> (
    SELECT hierarchy_path
    FROM graph_node
    WHERE entity_id = 'property-uuid' AND node_type = 'property'
)
ORDER BY nlevel(gn.hierarchy_path);
```

### Conflict Detection: "Are any of today's assignments conflicted?"

```sql
-- Find assignments where a cleaner has a CONFLICT edge with the client
SELECT
    j.job_number,
    c_cleaner.first_name || ' ' || c_cleaner.last_name AS cleaner_name,
    cl.display_name AS client_name,
    conflict_edge.properties->>'reason' AS conflict_reason,
    conflict_edge.weight AS severity
FROM job j,
     jsonb_array_elements(j.assignments) AS assignment
JOIN cleaner c_cleaner ON c_cleaner.id = (assignment->>'cleaner_id')::uuid
JOIN client cl ON cl.id = j.client_id
JOIN graph_node gn_cleaner ON gn_cleaner.entity_id = c_cleaner.id AND gn_cleaner.node_type = 'cleaner'
JOIN graph_node gn_client ON gn_client.entity_id = cl.id AND gn_client.node_type = 'client'
JOIN graph_edge conflict_edge ON conflict_edge.source_node_id = gn_cleaner.id
    AND conflict_edge.target_node_id = gn_client.id
    AND conflict_edge.edge_type = 'CONFLICT'
    AND conflict_edge.valid_until IS NULL
WHERE j.tenant_id = 'tenant-uuid'
  AND j.scheduled_start::date = CURRENT_DATE
  AND j.status IN ('scheduled', 'dispatched');
```

### Chemical Exposure Tracking: "Which cleaners have used this chemical?"

```sql
-- Graph traversal: chemical_product → (USES_PRODUCT) → job → assignment → cleaner
SELECT DISTINCT
    c.first_name, c.last_name,
    e.properties->>'job_date' AS used_on,
    e.properties->>'quantity' AS quantity_used
FROM graph_edge e
JOIN graph_node gn_product ON e.target_node_id = gn_product.id
    AND gn_product.node_type = 'chemical_product'
JOIN graph_node gn_job ON e.source_node_id = gn_job.id
    AND gn_job.node_type = 'job'
JOIN job j ON gn_job.entity_id = j.id,
     jsonb_array_elements(j.assignments) AS assignment
JOIN cleaner c ON c.id = (assignment->>'cleaner_id')::uuid
WHERE gn_product.entity_id = 'chemical-product-uuid'
  AND e.edge_type = 'USES_PRODUCT'
ORDER BY e.properties->>'job_date' DESC;
```

---

## Graph Maintenance: Triggers and Sync

```sql
-- Automatically create graph nodes when relational entities are created
-- Example: trigger on client INSERT

CREATE OR REPLACE FUNCTION sync_client_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties)
    VALUES (
        NEW.tenant_id,
        'client',
        NEW.id,
        NEW.display_name,
        jsonb_build_object(
            'is_company', NEW.is_company,
            'source', NEW.source
        )
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_client_graph_sync
    AFTER INSERT ON client
    FOR EACH ROW EXECUTE FUNCTION sync_client_to_graph();

-- Similar triggers for cleaner, property, contract, chemical_product, equipment
-- Update triggers maintain label and properties in sync
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Infrastructure | 2 | graph_node, graph_edge (the relationship engine) |
| Identity & Multi-Tenancy | 2 | tenant, user |
| Clients & Properties | 2 | client, property |
| Workforce | 1 | cleaner (skills/availability in JSONB) |
| Service Catalogue | 1 | service_type |
| Scheduling & Jobs | 2 | schedule_rule, job |
| Quality Inspection | 1 | inspection |
| Billing | 2 | invoice, payment |
| Chemical Compliance | 1 | chemical_product |
| Equipment | 1 | equipment |
| Contracts | 1 | contract |
| Routes | 1 | route |
| Communications | 1 | communication_log |
| **Total** | **~18** | Compact relational core + 2 graph tables that encode all relationships |

---

## Key Design Decisions

1. **graph_node / graph_edge as the universal relationship layer** — instead of dedicated junction tables for each relationship type (cleaner_skill, team_member, contract_property), all relationships live in `graph_edge` with an `edge_type` discriminator. This eliminates the proliferation of junction tables and enables relationship types to be added without schema changes.

2. **`ltree` for organisational hierarchies** — franchise, region, branch, and property hierarchies are modelled using PostgreSQL's native `ltree` type. This enables efficient subtree queries (`<@`), ancestor queries (`@>`), and depth calculations without recursive CTEs.

3. **Temporal edges with valid_from / valid_until** — every graph edge has a validity period. When a cleaner is reassigned from one property to another, the old edge gets a `valid_until` timestamp and a new edge is created. This preserves the full history of relationships without losing current state.

4. **Weight as a semantic measure** — the `weight` column on `graph_edge` has context-dependent meaning. For affinity edges, it represents strength of preference (0-1). For conflict edges, it represents severity. For referral edges, it can represent monetary value. The application layer interprets weight based on edge_type.

5. **Graph nodes mirror relational entities** — every client, cleaner, property, contract, and chemical product gets a corresponding `graph_node` row via database triggers. The `entity_id` foreign key links back to the relational table for full CRUD. The graph layer is purely additive.

6. **Relational core uses the hybrid JSONB pattern** — the operational tables (job, inspection, service_type) follow the Hybrid Relational + JSONB pattern from Model 3. The graph layer adds relationship intelligence on top without changing the operational schema.

7. **Conflict detection is a first-class edge type** — rather than maintaining a separate "do not assign" list, conflicts are graph edges with severity weights. The scheduling engine queries the graph before finalising assignments, and the system can distinguish between soft conflicts (preference) and hard blocks (legal/contractual).

8. **Cleaner-property affinity is computed and stored** — after each completed job, the application updates the affinity edge between the cleaner and the property, incorporating job count, inspection scores, and recency. The AI scheduling engine uses these edges for optimal assignment.

9. **Referral chain analysis without dedicated referral tables** — the REFERRED_BY edge type enables walking the full referral chain using recursive CTEs. Combined with payment data from the relational side, this calculates lifetime referral value per client without a separate referral tracking system.

10. **Graph layer is optional** — the relational core works independently. A deployment that does not need franchise hierarchies, affinity scoring, or conflict detection can simply not populate the graph tables. This makes the model suitable for both simple residential operations and complex multi-brand enterprises.
