# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Time & Attendance System · Created: 2026-05-12

## Philosophy

This model uses conventional relational tables for transactional CRUD operations (time entries, timesheets, schedules) and adds a property graph layer for relationship-heavy queries. The graph layer models organizational hierarchies, manager chains, location assignments, skill dependencies, compliance rule applicability, and shift coverage networks. These relationships are first-class entities that can be traversed, queried, and analyzed independently of the operational data.

Time and attendance systems have a hidden graph problem. When a manager approves a timesheet, the system must verify that the manager has authority over the employee — which requires traversing an organizational hierarchy. When the compliance engine checks overtime, it must determine which pay rules apply to an employee based on their location, department, jurisdiction, and employment type — a multi-hop relationship traversal. When the scheduling engine fills an open shift, it must find available employees with the right skills at the right location who won't exceed overtime thresholds — a constrained graph search.

In a traditional relational model, these queries require multiple self-joins, recursive CTEs, and complex WHERE clauses. In a graph-relational model, they become natural path traversals. The graph layer also enables AI-powered analytics: conflict-of-interest detection (manager approving their own spouse's timesheet), buddy-punching network analysis (which employees always clock in together?), and organizational bottleneck identification (which managers have the highest approval backlogs?).

**Best for:** Organizations with complex hierarchies, multi-site deployments, credential-matched scheduling, and a need for relationship-aware compliance and analytics.

**Trade-offs:**
- (+) Natural modeling of organizational hierarchies, authority chains, and skill networks
- (+) Efficient multi-hop queries: "who can approve this timesheet?" is a graph traversal, not a recursive CTE
- (+) AI-ready relationship data: anomaly detection, conflict-of-interest, network analysis
- (+) Flexible relationship types: add new relationship kinds without schema changes
- (+) Powerful for credential-matched scheduling: traverse skill-to-shift requirement graphs
- (-) Higher infrastructure complexity: two query languages/paradigms (SQL + graph traversal)
- (-) Data synchronization between relational and graph layers must be maintained
- (-) Fewer developers are experienced with graph databases or graph query patterns
- (-) Graph database licensing and operational costs (if using Neo4j/similar)
- (-) Potential data inconsistency if relational and graph layers diverge

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FLSA (29 CFR Part 778) | Pay rule applicability determined by graph traversal from employee through location/department/jurisdiction nodes to applicable pay rules |
| DCAA Timekeeping | Approval authority verified by traversing the manager-reports_to graph; DCAA requires supervisor approval — graph validates supervisor relationship |
| FMLA | Leave eligibility checked by traversing employee-to-jurisdiction edges to determine FMLA applicability (employer size, hours worked) |
| EU Working Time Directive | Compliance rule applicability mapped through jurisdiction graph; multi-country organizations traverse location-jurisdiction-rule paths |
| ISO 3166 | Jurisdiction hierarchy modeled as a graph: country nodes, subdivision nodes, with parent-child edges matching ISO 3166-1/3166-2 |
| GDPR / BIPA | Data access authorization verified by graph traversal of role-permission-scope paths; biometric data access restricted to authorized nodes |
| HR Open Standards | Employee-to-pay-rule-to-jurisdiction graph informs HR-JSON Timecard export field mapping |
| RFC 5545 (iCalendar) | Schedule-shift-employee graph enables efficient iCalendar feed generation per employee |

---

## Relational Tables (Operational CRUD)

```sql
-- ============================================================
-- TENANT
-- ============================================================
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    locale          VARCHAR(10) NOT NULL DEFAULT 'en-US',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- LOCATION
-- ============================================================
CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    timezone        VARCHAR(50) NOT NULL,
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    geofence_radius_meters INTEGER DEFAULT 100,
    geofence_polygon JSONB,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_location_tenant ON location (tenant_id);

-- ============================================================
-- DEPARTMENT
-- ============================================================
CREATE TABLE department (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    cost_center     VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- EMPLOYEE
-- ============================================================
CREATE TABLE employee (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_number VARCHAR(50) NOT NULL,
    email           VARCHAR(255),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    phone           VARCHAR(30),
    hire_date       DATE NOT NULL,
    termination_date DATE,
    employment_status VARCHAR(20) NOT NULL DEFAULT 'active',
    employment_type VARCHAR(20) NOT NULL DEFAULT 'full_time',
    flsa_status     VARCHAR(10) NOT NULL DEFAULT 'non_exempt',
    hourly_rate     NUMERIC(10,4),
    currency_code   CHAR(3) DEFAULT 'USD',
    timezone        VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, employee_number)
);

CREATE INDEX idx_employee_tenant ON employee (tenant_id);
CREATE INDEX idx_employee_status ON employee (tenant_id, employment_status);

-- ============================================================
-- SKILL
-- ============================================================
CREATE TABLE skill (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),
    requires_expiry BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- JURISDICTION
-- ============================================================
CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    name            VARCHAR(255) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PAY RULE
-- ============================================================
CREATE TABLE pay_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    workweek_start_day SMALLINT NOT NULL DEFAULT 0,
    overtime_weekly_threshold INTEGER NOT NULL DEFAULT 2400,
    overtime_daily_threshold INTEGER,
    overtime_multiplier NUMERIC(4,2) NOT NULL DEFAULT 1.50,
    doubletime_daily_threshold INTEGER,
    doubletime_multiplier NUMERIC(4,2) DEFAULT 2.00,
    max_weekly_hours INTEGER,
    min_daily_rest_minutes INTEGER,
    min_weekly_rest_minutes INTEGER,
    break_rules     JSONB NOT NULL DEFAULT '[]',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pay_rule_tenant ON pay_rule (tenant_id);

-- ============================================================
-- COMPLIANCE RULE
-- ============================================================
CREATE TABLE compliance_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    rule_type       VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    threshold_value INTEGER,
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning',
    regulation_reference VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Graph Layer (Relationship-Centric)

```sql
-- ============================================================
-- GRAPH NODE TABLE
-- ============================================================
-- Every entity that participates in relationships gets a node.
-- The node table is a lightweight registry; full entity data
-- stays in the relational tables above.
-- ============================================================

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY,                   -- same UUID as the entity's relational PK
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,               -- Employee, Location, Department, Skill,
                                                        -- Jurisdiction, PayRule, ComplianceRule, Role
    label           VARCHAR(255) NOT NULL,               -- human-readable label (employee name, location name)
    properties      JSONB NOT NULL DEFAULT '{}',         -- denormalized properties for graph queries
    -- Employee properties example:
    -- {
    --   "employee_number": "EMP-001",
    --   "employment_status": "active",
    --   "flsa_status": "non_exempt",
    --   "employment_type": "full_time"
    -- }
    -- Location properties example:
    -- {
    --   "country_code": "US",
    --   "subdivision_code": "US-CA",
    --   "timezone": "America/Los_Angeles"
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gn_tenant ON graph_node (tenant_id);
CREATE INDEX idx_gn_type ON graph_node (node_type);
CREATE INDEX idx_gn_props ON graph_node USING GIN (properties);

-- ============================================================
-- GRAPH EDGE TABLE
-- ============================================================
-- All relationships between entities are stored as edges.
-- Edges are directional, typed, and optionally temporal
-- (effective_from / effective_to for historical traversal).
-- ============================================================

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_id       UUID NOT NULL REFERENCES graph_node(id),
    target_id       UUID NOT NULL REFERENCES graph_node(id),
    edge_type       VARCHAR(50) NOT NULL,               -- see edge type catalogue below
    properties      JSONB NOT NULL DEFAULT '{}',         -- edge-specific metadata
    weight          NUMERIC(10,4) DEFAULT 1.0,           -- for weighted graph algorithms
    effective_from  TIMESTAMPTZ NOT NULL DEFAULT now(),   -- temporal: when this relationship started
    effective_to    TIMESTAMPTZ,                          -- NULL = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Core indexes for graph traversal
CREATE INDEX idx_ge_source ON graph_edge (source_id, edge_type) WHERE effective_to IS NULL;
CREATE INDEX idx_ge_target ON graph_edge (target_id, edge_type) WHERE effective_to IS NULL;
CREATE INDEX idx_ge_type ON graph_edge (edge_type) WHERE effective_to IS NULL;
CREATE INDEX idx_ge_tenant ON graph_edge (tenant_id);

-- Temporal index for historical queries
CREATE INDEX idx_ge_temporal ON graph_edge (source_id, effective_from, effective_to);

-- Prevent duplicate active edges of the same type
CREATE UNIQUE INDEX idx_ge_unique_active
    ON graph_edge (source_id, target_id, edge_type)
    WHERE effective_to IS NULL;
```

## Edge Type Catalogue

```
-- ============================================================
-- ORGANIZATIONAL RELATIONSHIPS
-- ============================================================
-- REPORTS_TO          Employee -> Employee (manager chain)
-- BELONGS_TO_DEPT     Employee -> Department
-- ASSIGNED_TO_LOC     Employee -> Location (can be multiple)
-- DEPT_IN_LOCATION    Department -> Location
-- DEPT_PARENT_OF      Department -> Department (hierarchy)

-- ============================================================
-- SKILL & QUALIFICATION RELATIONSHIPS
-- ============================================================
-- HAS_SKILL           Employee -> Skill
--   properties: { "acquired_date": "...", "expiry_date": "...",
--                  "credential_number": "CW-12345", "level": "expert" }
-- REQUIRES_SKILL      Shift -> Skill (shift requirement)
-- PREREQUISITE_OF     Skill -> Skill (skill dependencies)

-- ============================================================
-- PAY & COMPLIANCE RELATIONSHIPS
-- ============================================================
-- GOVERNED_BY         Location -> Jurisdiction
-- SUBJECT_TO          Employee -> PayRule
--   properties: { "hourly_rate": 25.00, "currency": "USD" }
-- APPLIES_IN          PayRule -> Jurisdiction
-- ENFORCED_BY         Jurisdiction -> ComplianceRule
-- EXEMPT_FROM         Employee -> ComplianceRule (exemptions)

-- ============================================================
-- AUTHORIZATION RELATIONSHIPS
-- ============================================================
-- HAS_ROLE            Employee -> Role
--   properties: { "scope_type": "location", "scope_id": "uuid" }
-- CAN_APPROVE         Employee -> Employee (approval authority)
--   properties: { "approval_types": ["timesheet", "leave", "shift_swap"] }

-- ============================================================
-- SCHEDULING RELATIONSHIPS
-- ============================================================
-- AVAILABLE_AT        Employee -> Location (availability)
--   properties: { "day_of_week": 1, "from": "06:00", "to": "22:00" }
-- PREFERS             Employee -> Shift pattern preferences
--   properties: { "shift_type": "morning", "strength": 0.8 }
-- PAIRED_WITH         Employee -> Employee (co-scheduling preferences)
--   properties: { "reason": "training_pair", "mandatory": true }

-- ============================================================
-- TEMPORAL / AUDIT RELATIONSHIPS
-- ============================================================
-- APPROVED_BY         Timesheet -> Employee (who approved)
-- SUPERVISED_BY       TimeEntry -> Employee (supervisor at time of entry)
```

## Edge Properties Examples

```sql
-- ============================================================
-- Example: Manager reporting chain
-- ============================================================
INSERT INTO graph_edge (tenant_id, source_id, target_id, edge_type, properties)
VALUES (
    '{{tenant_uuid}}',
    '{{employee_alice_uuid}}',     -- Alice
    '{{employee_bob_uuid}}',       -- Bob (Alice's manager)
    'REPORTS_TO',
    '{"direct": true, "delegation_active": false}'
);

-- ============================================================
-- Example: Employee skill with expiry
-- ============================================================
INSERT INTO graph_edge (tenant_id, source_id, target_id, edge_type, properties)
VALUES (
    '{{tenant_uuid}}',
    '{{employee_uuid}}',
    '{{skill_forklift_uuid}}',
    'HAS_SKILL',
    '{
        "acquired_date": "2024-06-15",
        "expiry_date": "2026-06-15",
        "credential_number": "FL-2024-789",
        "level": "certified",
        "verified_by": "uuid",
        "verified_at": "2024-06-15T10:30:00Z"
    }'
);

-- ============================================================
-- Example: Employee subject to California pay rules
-- ============================================================
INSERT INTO graph_edge (tenant_id, source_id, target_id, edge_type, properties)
VALUES (
    '{{tenant_uuid}}',
    '{{employee_uuid}}',
    '{{pay_rule_california_uuid}}',
    'SUBJECT_TO',
    '{
        "hourly_rate": 28.50,
        "currency": "USD",
        "effective_date": "2026-01-01"
    }'
);
```

## Operational Tables (Time, Scheduling, Leave)

```sql
-- ============================================================
-- TIME ENTRY
-- ============================================================
CREATE TABLE time_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    entry_type      VARCHAR(20) NOT NULL,
    punch_time      TIMESTAMPTZ NOT NULL,
    adjusted_time   TIMESTAMPTZ,
    source          VARCHAR(30) NOT NULL,
    location_id     UUID REFERENCES location(id),
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    geofence_status VARCHAR(20),
    biometric_method VARCHAR(30),
    biometric_confidence NUMERIC(5,3),
    is_flagged      BOOLEAN NOT NULL DEFAULT false,
    flag_reason     VARCHAR(100),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_te_employee ON time_entry (employee_id, punch_time);
CREATE INDEX idx_te_tenant ON time_entry (tenant_id, punch_time);
CREATE INDEX idx_te_flagged ON time_entry (tenant_id) WHERE is_flagged = true;

-- ============================================================
-- TIME ENTRY AUDIT
-- ============================================================
CREATE TABLE time_entry_audit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    time_entry_id   UUID NOT NULL REFERENCES time_entry(id),
    field_name      VARCHAR(50) NOT NULL,
    old_value       TEXT,
    new_value       TEXT,
    changed_by      UUID NOT NULL REFERENCES employee(id),
    change_reason   TEXT,
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tea_entry ON time_entry_audit (time_entry_id);

-- ============================================================
-- TIMESHEET
-- ============================================================
CREATE TABLE timesheet (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    period_type     VARCHAR(10) NOT NULL DEFAULT 'weekly',
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    total_regular_minutes   INTEGER NOT NULL DEFAULT 0,
    total_overtime_minutes  INTEGER NOT NULL DEFAULT 0,
    total_doubletime_minutes INTEGER NOT NULL DEFAULT 0,
    total_break_minutes     INTEGER NOT NULL DEFAULT 0,
    total_pto_minutes       INTEGER NOT NULL DEFAULT 0,
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    approved_by     UUID REFERENCES employee(id),
    rejected_at     TIMESTAMPTZ,
    rejection_reason TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, period_start, period_type)
);

CREATE INDEX idx_ts_employee ON timesheet (employee_id, period_start);
CREATE INDEX idx_ts_status ON timesheet (tenant_id, status);

-- ============================================================
-- TIMESHEET LINE
-- ============================================================
CREATE TABLE timesheet_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timesheet_id    UUID NOT NULL REFERENCES timesheet(id) ON DELETE CASCADE,
    work_date       DATE NOT NULL,
    location_id     UUID REFERENCES location(id),
    department_id   UUID REFERENCES department(id),
    project_code    VARCHAR(50),
    labor_category  VARCHAR(50),
    regular_minutes INTEGER NOT NULL DEFAULT 0,
    overtime_minutes INTEGER NOT NULL DEFAULT 0,
    doubletime_minutes INTEGER NOT NULL DEFAULT 0,
    break_minutes   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tsl_timesheet ON timesheet_line (timesheet_id);

-- ============================================================
-- SCHEDULE
-- ============================================================
CREATE TABLE schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    name            VARCHAR(255) NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_schedule_location ON schedule (location_id, period_start);

-- ============================================================
-- SHIFT
-- ============================================================
CREATE TABLE shift (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schedule_id     UUID NOT NULL REFERENCES schedule(id) ON DELETE CASCADE,
    employee_id     UUID REFERENCES employee(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    department_id   UUID REFERENCES department(id),
    shift_date      DATE NOT NULL,
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    break_minutes   INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'scheduled',
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shift_employee ON shift (employee_id, shift_date);
CREATE INDEX idx_shift_location ON shift (location_id, shift_date);
CREATE INDEX idx_shift_open ON shift (schedule_id) WHERE employee_id IS NULL;

-- ============================================================
-- LEAVE REQUEST
-- ============================================================
CREATE TABLE leave_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    leave_type      VARCHAR(50) NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    total_hours     NUMERIC(10,2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    reason          TEXT,
    approved_by     UUID REFERENCES employee(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lr_employee ON leave_request (employee_id, start_date);

-- ============================================================
-- COMPLIANCE ALERT
-- ============================================================
CREATE TABLE compliance_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    compliance_rule_id UUID NOT NULL REFERENCES compliance_rule(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    alert_type      VARCHAR(30) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    message         TEXT NOT NULL,
    context_data    JSONB,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    acknowledged_by UUID REFERENCES employee(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ca_tenant ON compliance_alert (tenant_id, status);
CREATE INDEX idx_ca_employee ON compliance_alert (employee_id);

-- ============================================================
-- BIOMETRIC TEMPLATE
-- ============================================================
CREATE TABLE biometric_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    biometric_type  VARCHAR(30) NOT NULL,
    template_data   BYTEA NOT NULL,
    algorithm_name  VARCHAR(100) NOT NULL,
    algorithm_version VARCHAR(20) NOT NULL,
    quality_score   NUMERIC(5,3),
    encryption_key_id VARCHAR(100) NOT NULL,
    scheduled_destruction_date DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bt_employee ON biometric_template (employee_id);

-- ============================================================
-- BIOMETRIC CONSENT
-- ============================================================
CREATE TABLE biometric_consent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    consent_type    VARCHAR(30) NOT NULL,
    purpose         TEXT NOT NULL,
    retention_period_days INTEGER NOT NULL,
    consent_given   BOOLEAN NOT NULL,
    consent_text    TEXT NOT NULL,
    jurisdiction_law VARCHAR(50),
    given_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bc_employee ON biometric_consent (employee_id);
```

## Graph Query Examples (SQL with Recursive CTEs)

```sql
-- ============================================================
-- QUERY 1: Find the full management chain for an employee
-- (Who are all the managers above Alice, up to the CEO?)
-- ============================================================
WITH RECURSIVE management_chain AS (
    -- Start with the employee
    SELECT
        e.source_id AS employee_id,
        e.target_id AS manager_id,
        gn.label AS manager_name,
        1 AS depth
    FROM graph_edge e
    JOIN graph_node gn ON e.target_id = gn.id
    WHERE e.source_id = '{{alice_uuid}}'
      AND e.edge_type = 'REPORTS_TO'
      AND e.effective_to IS NULL

    UNION ALL

    -- Traverse up the chain
    SELECT
        mc.manager_id AS employee_id,
        e.target_id AS manager_id,
        gn.label AS manager_name,
        mc.depth + 1
    FROM management_chain mc
    JOIN graph_edge e ON mc.manager_id = e.source_id
        AND e.edge_type = 'REPORTS_TO'
        AND e.effective_to IS NULL
    JOIN graph_node gn ON e.target_id = gn.id
    WHERE mc.depth < 10  -- prevent infinite loops
)
SELECT * FROM management_chain ORDER BY depth;

-- ============================================================
-- QUERY 2: Find all employees who can fill an open shift
-- (Must have required skill, be assigned to the location,
--  be available on the day, and not exceed overtime threshold)
-- ============================================================
WITH shift_requirements AS (
    SELECT
        s.id AS shift_id,
        s.location_id,
        s.shift_date,
        s.start_time,
        s.end_time,
        EXTRACT(EPOCH FROM (s.end_time - s.start_time)) / 60 AS shift_minutes,
        ge.target_id AS required_skill_id
    FROM shift s
    LEFT JOIN graph_edge ge ON ge.source_id = s.id
        AND ge.edge_type = 'REQUIRES_SKILL'
        AND ge.effective_to IS NULL
    WHERE s.id = '{{open_shift_uuid}}'
),
eligible_employees AS (
    SELECT DISTINCT e.id AS employee_id, gn.label AS employee_name
    FROM employee e
    JOIN graph_node gn ON e.id = gn.id
    -- Must be assigned to this location
    JOIN graph_edge loc_edge ON loc_edge.source_id = e.id
        AND loc_edge.edge_type = 'ASSIGNED_TO_LOC'
        AND loc_edge.target_id = (SELECT location_id FROM shift_requirements)
        AND loc_edge.effective_to IS NULL
    -- Must have required skill (if any)
    LEFT JOIN shift_requirements sr ON true
    LEFT JOIN graph_edge skill_edge ON skill_edge.source_id = e.id
        AND skill_edge.edge_type = 'HAS_SKILL'
        AND skill_edge.target_id = sr.required_skill_id
        AND skill_edge.effective_to IS NULL
        AND (skill_edge.properties->>'expiry_date' IS NULL
             OR (skill_edge.properties->>'expiry_date')::date > sr.shift_date)
    WHERE e.employment_status = 'active'
      AND (sr.required_skill_id IS NULL OR skill_edge.id IS NOT NULL)
),
current_weekly_hours AS (
    SELECT
        te.employee_id,
        SUM(EXTRACT(EPOCH FROM (
            LEAD(te.punch_time) OVER (PARTITION BY te.employee_id ORDER BY te.punch_time)
            - te.punch_time
        )) / 60) AS worked_minutes_this_week
    FROM time_entry te
    WHERE te.employee_id IN (SELECT employee_id FROM eligible_employees)
      AND te.entry_type = 'clock_in'
      AND te.punch_time >= date_trunc('week', (SELECT shift_date FROM shift_requirements)::timestamp)
    GROUP BY te.employee_id
)
SELECT
    ee.employee_id,
    ee.employee_name,
    COALESCE(cwh.worked_minutes_this_week, 0) AS current_weekly_minutes,
    pr.overtime_weekly_threshold - COALESCE(cwh.worked_minutes_this_week, 0) AS remaining_before_ot
FROM eligible_employees ee
LEFT JOIN current_weekly_hours cwh ON ee.employee_id = cwh.employee_id
-- Get applicable pay rule via graph
JOIN graph_edge pr_edge ON pr_edge.source_id = ee.employee_id
    AND pr_edge.edge_type = 'SUBJECT_TO'
    AND pr_edge.effective_to IS NULL
JOIN pay_rule pr ON pr_edge.target_id = pr.id
WHERE COALESCE(cwh.worked_minutes_this_week, 0) +
      (SELECT shift_minutes FROM shift_requirements) <= pr.overtime_weekly_threshold
ORDER BY remaining_before_ot DESC;

-- ============================================================
-- QUERY 3: Validate approval authority
-- (Can this manager approve this employee's timesheet?)
-- ============================================================
WITH RECURSIVE authority_chain AS (
    -- Direct reports
    SELECT
        target_id AS manager_id,
        source_id AS subordinate_id,
        1 AS depth
    FROM graph_edge
    WHERE edge_type = 'REPORTS_TO'
      AND target_id = '{{manager_uuid}}'
      AND effective_to IS NULL

    UNION ALL

    -- Indirect reports (skip-level authority)
    SELECT
        ac.manager_id,
        e.source_id AS subordinate_id,
        ac.depth + 1
    FROM authority_chain ac
    JOIN graph_edge e ON e.target_id = ac.subordinate_id
        AND e.edge_type = 'REPORTS_TO'
        AND e.effective_to IS NULL
    WHERE ac.depth < 5  -- limit traversal depth
)
SELECT EXISTS (
    SELECT 1 FROM authority_chain
    WHERE subordinate_id = '{{employee_uuid}}'
) AS has_authority;

-- ============================================================
-- QUERY 4: Conflict of interest detection
-- (Find managers who approved timesheets for family members)
-- ============================================================
SELECT
    t.id AS timesheet_id,
    emp_node.label AS employee_name,
    mgr_node.label AS approver_name,
    rel.edge_type AS relationship,
    rel.properties->>'relation' AS relation_type
FROM timesheet t
JOIN graph_node emp_node ON t.employee_id = emp_node.id
JOIN graph_node mgr_node ON t.approved_by = mgr_node.id
JOIN graph_edge rel ON (
    (rel.source_id = t.employee_id AND rel.target_id = t.approved_by)
    OR (rel.source_id = t.approved_by AND rel.target_id = t.employee_id)
)
WHERE rel.edge_type = 'PAIRED_WITH'
  AND rel.properties->>'reason' = 'family_member'
  AND rel.effective_to IS NULL
  AND t.status = 'approved';

-- ============================================================
-- QUERY 5: Find employees with expiring certifications
-- assigned to shifts that require those certifications
-- ============================================================
SELECT
    emp_node.label AS employee_name,
    skill_node.label AS certification,
    (skill_edge.properties->>'expiry_date')::date AS expires,
    s.shift_date,
    s.start_time
FROM graph_edge skill_edge
JOIN graph_node emp_node ON skill_edge.source_id = emp_node.id
JOIN graph_node skill_node ON skill_edge.target_id = skill_node.id
JOIN graph_edge shift_req ON shift_req.target_id = skill_edge.target_id
    AND shift_req.edge_type = 'REQUIRES_SKILL'
    AND shift_req.effective_to IS NULL
JOIN shift s ON s.id = shift_req.source_id
    AND s.employee_id = skill_edge.source_id
WHERE skill_edge.edge_type = 'HAS_SKILL'
  AND skill_edge.effective_to IS NULL
  AND (skill_edge.properties->>'expiry_date')::date BETWEEN now() AND now() + INTERVAL '90 days'
  AND s.shift_date > now()::date
ORDER BY (skill_edge.properties->>'expiry_date')::date;

-- ============================================================
-- QUERY 6: Organizational structure snapshot at a past date
-- (What did the reporting structure look like on Jan 1, 2026?)
-- ============================================================
SELECT
    source_node.label AS employee,
    target_node.label AS manager,
    e.properties,
    e.effective_from,
    e.effective_to
FROM graph_edge e
JOIN graph_node source_node ON e.source_id = source_node.id
JOIN graph_node target_node ON e.target_id = target_node.id
WHERE e.edge_type = 'REPORTS_TO'
  AND e.tenant_id = '{{tenant_uuid}}'
  AND e.effective_from <= '2026-01-01'
  AND (e.effective_to IS NULL OR e.effective_to > '2026-01-01')
ORDER BY target_node.label, source_node.label;
```

## Graph Synchronization

```sql
-- ============================================================
-- TRIGGER: Sync employee creation to graph_node
-- ============================================================
-- When a new employee is created in the relational table,
-- automatically create a corresponding graph_node.
-- ============================================================

CREATE OR REPLACE FUNCTION sync_employee_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_node (id, tenant_id, node_type, label, properties)
    VALUES (
        NEW.id,
        NEW.tenant_id,
        'Employee',
        NEW.first_name || ' ' || NEW.last_name,
        jsonb_build_object(
            'employee_number', NEW.employee_number,
            'employment_status', NEW.employment_status,
            'flsa_status', NEW.flsa_status,
            'employment_type', NEW.employment_type
        )
    )
    ON CONFLICT (id) DO UPDATE SET
        label = EXCLUDED.label,
        properties = EXCLUDED.properties,
        is_active = (NEW.employment_status != 'terminated'),
        updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_employee_to_graph
    AFTER INSERT OR UPDATE ON employee
    FOR EACH ROW EXECUTE FUNCTION sync_employee_to_graph();

-- ============================================================
-- Similar triggers for location, department, skill, jurisdiction,
-- pay_rule, and compliance_rule tables.
-- ============================================================

CREATE OR REPLACE FUNCTION sync_location_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_node (id, tenant_id, node_type, label, properties)
    VALUES (
        NEW.id,
        NEW.tenant_id,
        'Location',
        NEW.name,
        jsonb_build_object(
            'country_code', NEW.country_code,
            'subdivision_code', NEW.subdivision_code,
            'timezone', NEW.timezone
        )
    )
    ON CONFLICT (id) DO UPDATE SET
        label = EXCLUDED.label,
        properties = EXCLUDED.properties,
        is_active = NEW.is_active,
        updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_location_to_graph
    AFTER INSERT OR UPDATE ON location
    FOR EACH ROW EXECUTE FUNCTION sync_location_to_graph();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge (replaces 8+ junction/relationship tables) |
| Tenant & Organization | 5 | tenant, location, department, jurisdiction, skill |
| Employee | 1 | employee (relationships in graph layer) |
| Time Capture | 3 | time_entry, time_entry_audit, timesheet + timesheet_line |
| Scheduling | 3 | schedule, shift, leave_request |
| Pay & Compliance | 4 | pay_rule, compliance_rule, compliance_alert |
| Biometric | 2 | biometric_template, biometric_consent |
| **Total** | **~21** | Graph layer absorbs junction tables and relationship complexity |

---

## Key Design Decisions

1. **Two-table graph model (graph_node + graph_edge).** All entity relationships are stored as edges between nodes. This replaces employee_location, employee_skill, employee_role, department_hierarchy, and other junction tables with a single, flexible edge table. New relationship types are added by inserting edges with new `edge_type` values, not by creating new tables.

2. **Temporal edges for historical analysis.** Every edge has `effective_from` and `effective_to` timestamps, enabling point-in-time organizational queries ("who reported to whom on January 1?"). This is critical for compliance: DCAA audits may ask about approval authority at the time a timesheet was approved, not today.

3. **Graph implemented in PostgreSQL, not a separate database.** The graph_node and graph_edge tables live in the same PostgreSQL database as the operational tables. This avoids the operational complexity of a separate graph database (Neo4j) while providing graph traversal via recursive CTEs. If graph query volume grows, the model can be migrated to Apache AGE (PostgreSQL graph extension) or Neo4j without changing the relational tables.

4. **Denormalized properties on graph nodes.** Graph nodes carry a JSONB `properties` column with key attributes copied from the relational entity. This enables graph-only queries without joining back to relational tables for simple traversals. Properties are kept in sync via database triggers.

5. **Trigger-based synchronization.** Relational INSERT/UPDATE triggers automatically maintain the graph_node table. This ensures the graph layer is always consistent with the relational source of truth. Edge creation/modification is handled at the application layer since edges represent domain-specific relationships that require business logic.

6. **Weighted edges for AI scheduling optimization.** The `weight` column on graph_edge enables weighted graph algorithms. Employee-location assignment weights can reflect commute distance or preference strength. Skill-employee weights can reflect proficiency level. The scheduling AI can use these weights to optimize shift assignments.

7. **Approval authority as graph traversal.** Instead of hardcoding approval logic (direct manager only), the graph supports flexible authority patterns: direct manager, skip-level approval, delegated authority, and cross-department approval — all as graph traversal queries. Adding a new approval authority is inserting an edge, not changing code.

8. **Conflict-of-interest detection via relationship analysis.** The graph layer enables queries that are impractical in a relational model: finding all cases where a manager approved a timesheet for someone they have a personal relationship with. This is a natural graph pattern-matching problem.

9. **Credential-matched scheduling as constrained graph search.** Finding employees eligible for an open shift (has skill + at location + available + under OT threshold) is a multi-hop graph traversal with constraints. The graph model makes this query natural and extensible.

10. **Relational tables remain the system of record.** The graph layer is a derived structure that enriches relationship queries. If the graph layer fails or becomes inconsistent, the relational tables contain all data needed to rebuild it. This provides a safety net that a graph-only architecture would not have.
