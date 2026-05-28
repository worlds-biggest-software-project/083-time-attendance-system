# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Time & Attendance System · Created: 2026-05-12

## Philosophy

This model follows classical relational database design with full normalization (3NF+). Every domain concept — employees, locations, shifts, punches, pay rules, leave types, compliance rules — gets its own table with strong foreign key constraints. The schema is designed for referential integrity and complex cross-entity queries, making it ideal for environments where data correctness is non-negotiable (FLSA audits, DCAA timekeeping, BIPA compliance).

The approach mirrors how established workforce management platforms like UKG and Replicon structure their data internally: separate tables for every entity, junction tables for many-to-many relationships, and lookup tables for reference data. This is the most predictable architecture for teams with SQL expertise and produces the most query-friendly schema for compliance reporting.

The trade-off is table count and migration complexity. Adding a new jurisdiction's pay rules or a new leave type requires schema changes (new rows in reference tables, potentially new columns). But the benefit is that every relationship is explicit, every constraint is enforced at the database level, and every query can be optimized with standard indexing.

**Best for:** Compliance-heavy deployments where FLSA/DCAA audit readiness, referential integrity, and complex cross-entity reporting are the top priorities.

**Trade-offs:**
- (+) Strongest data integrity — all relationships enforced by foreign keys
- (+) Most query-friendly for compliance reports and payroll exports
- (+) Well-understood by most development teams; abundant tooling
- (+) Easiest to reason about for auditors and regulators
- (-) Highest table count (~45-55 tables); more migration overhead
- (-) Adding jurisdiction-specific fields requires schema changes
- (-) Less flexible for rapid prototyping or variable data shapes
- (-) Join-heavy queries can become complex for cross-domain reports

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FLSA (29 CFR Part 778) | `pay_rule` and `overtime_rule` tables encode FLSA overtime thresholds (40 hrs/week), regular rate calculation, and exempt/non-exempt classification per employee |
| FMLA | `leave_type` and `leave_entitlement` tables track protected absence categories, intermittent leave balances, and eligibility rules |
| EU Working Time Directive (2003/88/EC) | `compliance_rule` table stores maximum weekly hours (48), minimum rest periods (11h daily, 24h weekly), and jurisdiction-specific implementations |
| DCAA Timekeeping | `time_entry` table includes `project_code`, `labor_category`, `recorded_at` (contemporaneous timestamp), and `supervisor_approval_id` for audit trail |
| ISO/IEC 19794-5 | `biometric_template` table stores facial image data in standards-compliant format with `template_standard` field |
| GDPR Article 9 / Illinois BIPA | `biometric_consent` table tracks explicit consent records with purpose, retention schedule, and destruction date |
| RFC 5545 (iCalendar) | `shift` and `schedule` tables include fields compatible with iCalendar VEVENT export (DTSTART, DTEND, RRULE, UID) |
| HR Open Standards (HR-JSON Timecard) | `time_entry` schema maps directly to HR-JSON Timecard fields for interoperable payroll export |
| ISO 3166 | `jurisdiction` table uses ISO 3166-1 alpha-2 country codes and ISO 3166-2 subdivision codes |
| OAuth 2.0 / OpenID Connect | `oauth_client` and `user_session` tables support API authorization and SSO integration |

---

## Core Identity & Multi-Tenancy

```sql
-- ============================================================
-- TENANT / ORGANIZATION
-- ============================================================
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,     -- URL-safe identifier
    subscription_plan VARCHAR(50) NOT NULL DEFAULT 'free',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    locale          VARCHAR(10) NOT NULL DEFAULT 'en-US',
    settings        JSONB NOT NULL DEFAULT '{}',       -- tenant-level config overrides
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenant_slug ON tenant (slug);

-- ============================================================
-- LOCATION / WORK SITE
-- ============================================================
CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL,                  -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),                        -- ISO 3166-2
    timezone        VARCHAR(50) NOT NULL,
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    geofence_radius_meters INTEGER DEFAULT 100,         -- circular geofence
    geofence_polygon JSONB,                             -- GeoJSON polygon for complex fences
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_location_tenant ON location (tenant_id);
CREATE INDEX idx_location_country ON location (country_code);

-- ============================================================
-- DEPARTMENT
-- ============================================================
CREATE TABLE department (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    location_id     UUID REFERENCES location(id),
    parent_id       UUID REFERENCES department(id),    -- hierarchical departments
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    cost_center     VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_department_tenant ON department (tenant_id);
CREATE INDEX idx_department_parent ON department (parent_id);

-- ============================================================
-- JURISDICTION (pay rule & compliance context)
-- ============================================================
CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,                  -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),                        -- ISO 3166-2 (e.g., US-CA for California)
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_jurisdiction_codes ON jurisdiction (country_code, subdivision_code);
```

## Employee & Role Management

```sql
-- ============================================================
-- EMPLOYEE
-- ============================================================
CREATE TABLE employee (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_number VARCHAR(50) NOT NULL,               -- employer-assigned ID
    email           VARCHAR(255),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    phone           VARCHAR(30),
    date_of_birth   DATE,
    hire_date       DATE NOT NULL,
    termination_date DATE,
    employment_status VARCHAR(20) NOT NULL DEFAULT 'active',  -- active, on_leave, terminated, suspended
    employment_type VARCHAR(20) NOT NULL DEFAULT 'full_time', -- full_time, part_time, contractor, temp
    flsa_status     VARCHAR(10) NOT NULL DEFAULT 'non_exempt', -- exempt, non_exempt
    primary_location_id UUID REFERENCES location(id),
    primary_department_id UUID REFERENCES department(id),
    jurisdiction_id UUID REFERENCES jurisdiction(id),   -- determines pay rules and compliance
    manager_id      UUID REFERENCES employee(id),
    timezone        VARCHAR(50),                        -- override; falls back to location timezone
    avatar_url      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, employee_number)
);

CREATE INDEX idx_employee_tenant ON employee (tenant_id);
CREATE INDEX idx_employee_status ON employee (tenant_id, employment_status);
CREATE INDEX idx_employee_manager ON employee (manager_id);
CREATE INDEX idx_employee_location ON employee (primary_location_id);
CREATE INDEX idx_employee_department ON employee (primary_department_id);

-- ============================================================
-- EMPLOYEE LOCATION ASSIGNMENT (many-to-many)
-- ============================================================
CREATE TABLE employee_location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    effective_from  DATE NOT NULL,
    effective_to    DATE,                               -- NULL = current
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, location_id, effective_from)
);

CREATE INDEX idx_emp_location_employee ON employee_location (employee_id);

-- ============================================================
-- SKILL / CERTIFICATION
-- ============================================================
CREATE TABLE skill (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),                       -- e.g., 'certification', 'license', 'training'
    requires_expiry BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE employee_skill (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    skill_id        UUID NOT NULL REFERENCES skill(id),
    acquired_date   DATE,
    expiry_date     DATE,
    credential_number VARCHAR(100),
    verified_by     UUID REFERENCES employee(id),
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, skill_id, acquired_date)
);

CREATE INDEX idx_emp_skill_employee ON employee_skill (employee_id);
CREATE INDEX idx_emp_skill_expiry ON employee_skill (expiry_date) WHERE expiry_date IS NOT NULL;

-- ============================================================
-- ROLE & PERMISSION (RBAC)
-- ============================================================
CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT false,     -- built-in roles (admin, manager, employee)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,       -- e.g., 'timesheet.approve', 'schedule.edit'
    description     TEXT,
    category        VARCHAR(50)                         -- e.g., 'timesheet', 'schedule', 'admin'
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id),
    permission_id   UUID NOT NULL REFERENCES permission(id),
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE employee_role (
    employee_id     UUID NOT NULL REFERENCES employee(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    scope_type      VARCHAR(20) NOT NULL DEFAULT 'global', -- global, location, department
    scope_id        UUID,                               -- location_id or department_id if scoped
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES employee(id),
    PRIMARY KEY (employee_id, role_id, scope_type, COALESCE(scope_id, '00000000-0000-0000-0000-000000000000'))
);

CREATE INDEX idx_emp_role_employee ON employee_role (employee_id);
```

## Time Capture & Punches

```sql
-- ============================================================
-- TIME ENTRY (punch / clock event)
-- ============================================================
CREATE TABLE time_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    entry_type      VARCHAR(20) NOT NULL,               -- clock_in, clock_out, break_start, break_end
    punch_time      TIMESTAMPTZ NOT NULL,               -- actual recorded time
    adjusted_time   TIMESTAMPTZ,                        -- manager-adjusted time (if rounding/correction)
    source          VARCHAR(30) NOT NULL,               -- web, mobile_app, kiosk, hardware_clock, manual, api
    location_id     UUID REFERENCES location(id),
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    geofence_status VARCHAR(20),                        -- inside, outside, unknown
    ip_address      INET,
    device_id       VARCHAR(255),
    biometric_method VARCHAR(30),                       -- facial_recognition, fingerprint, none
    biometric_confidence NUMERIC(5,3),                  -- match confidence score (0.000-1.000)
    notes           TEXT,
    is_flagged       BOOLEAN NOT NULL DEFAULT false,    -- anomaly detected
    flag_reason     VARCHAR(100),                       -- e.g., 'outside_geofence', 'low_biometric_confidence'
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(), -- DCAA: contemporaneous recording timestamp
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_time_entry_employee_date ON time_entry (employee_id, punch_time);
CREATE INDEX idx_time_entry_tenant_date ON time_entry (tenant_id, punch_time);
CREATE INDEX idx_time_entry_flagged ON time_entry (tenant_id, is_flagged) WHERE is_flagged = true;
CREATE INDEX idx_time_entry_source ON time_entry (source);

-- ============================================================
-- TIMESHEET (daily or weekly aggregation)
-- ============================================================
CREATE TABLE timesheet (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    period_type     VARCHAR(10) NOT NULL DEFAULT 'weekly', -- daily, weekly, biweekly, monthly
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, submitted, approved, rejected, locked
    total_regular_minutes   INTEGER NOT NULL DEFAULT 0,
    total_overtime_minutes  INTEGER NOT NULL DEFAULT 0,
    total_doubletime_minutes INTEGER NOT NULL DEFAULT 0,
    total_break_minutes     INTEGER NOT NULL DEFAULT 0,
    total_pto_minutes       INTEGER NOT NULL DEFAULT 0,
    submitted_at    TIMESTAMPTZ,
    submitted_by    UUID REFERENCES employee(id),
    approved_at     TIMESTAMPTZ,
    approved_by     UUID REFERENCES employee(id),
    rejected_at     TIMESTAMPTZ,
    rejected_by     UUID REFERENCES employee(id),
    rejection_reason TEXT,
    locked_at       TIMESTAMPTZ,
    locked_by       UUID REFERENCES employee(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, period_start, period_type)
);

CREATE INDEX idx_timesheet_employee ON timesheet (employee_id, period_start);
CREATE INDEX idx_timesheet_status ON timesheet (tenant_id, status);
CREATE INDEX idx_timesheet_approval ON timesheet (approved_by) WHERE approved_by IS NOT NULL;

-- ============================================================
-- TIMESHEET LINE (daily detail within a timesheet)
-- ============================================================
CREATE TABLE timesheet_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timesheet_id    UUID NOT NULL REFERENCES timesheet(id) ON DELETE CASCADE,
    work_date       DATE NOT NULL,
    location_id     UUID REFERENCES location(id),
    department_id   UUID REFERENCES department(id),
    project_code    VARCHAR(50),                        -- DCAA: project/contract allocation
    labor_category  VARCHAR(50),                        -- DCAA: labor category
    regular_minutes INTEGER NOT NULL DEFAULT 0,
    overtime_minutes INTEGER NOT NULL DEFAULT 0,
    doubletime_minutes INTEGER NOT NULL DEFAULT 0,
    break_minutes   INTEGER NOT NULL DEFAULT 0,
    first_clock_in  TIMESTAMPTZ,
    last_clock_out  TIMESTAMPTZ,
    is_holiday      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (timesheet_id, work_date, COALESCE(project_code, '__none__'))
);

CREATE INDEX idx_timesheet_line_sheet ON timesheet_line (timesheet_id);
CREATE INDEX idx_timesheet_line_date ON timesheet_line (work_date);

-- ============================================================
-- TIME ENTRY EDIT HISTORY (immutable audit trail)
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

CREATE INDEX idx_time_audit_entry ON time_entry_audit (time_entry_id);
CREATE INDEX idx_time_audit_changed_by ON time_entry_audit (changed_by);
```

## Pay Rules & Overtime

```sql
-- ============================================================
-- PAY RULE (overtime and premium pay configuration)
-- ============================================================
CREATE TABLE pay_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    workweek_start_day SMALLINT NOT NULL DEFAULT 0,     -- 0=Sunday, 1=Monday, ...6=Saturday
    overtime_weekly_threshold INTEGER NOT NULL DEFAULT 2400, -- minutes (40 hours = 2400)
    overtime_daily_threshold INTEGER,                    -- minutes; NULL = no daily OT (California: 480)
    overtime_multiplier NUMERIC(4,2) NOT NULL DEFAULT 1.50, -- FLSA standard: 1.5x
    doubletime_daily_threshold INTEGER,                  -- minutes; NULL = no DT (California: 720 = 12 hrs)
    doubletime_multiplier NUMERIC(4,2) DEFAULT 2.00,
    seventh_day_overtime BOOLEAN NOT NULL DEFAULT false,  -- California: OT on 7th consecutive day
    max_weekly_hours INTEGER,                            -- EU WTD: 2880 minutes (48 hours)
    min_daily_rest_minutes INTEGER,                      -- EU WTD: 660 minutes (11 hours)
    min_weekly_rest_minutes INTEGER,                     -- EU WTD: 1440 minutes (24 hours)
    break_rules     JSONB NOT NULL DEFAULT '[]',
    -- Example break_rules: [{"after_minutes": 300, "min_break_minutes": 30, "paid": false}]
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pay_rule_tenant ON pay_rule (tenant_id);
CREATE INDEX idx_pay_rule_jurisdiction ON pay_rule (jurisdiction_id);

-- ============================================================
-- EMPLOYEE PAY ASSIGNMENT
-- ============================================================
CREATE TABLE employee_pay_assignment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    pay_rule_id     UUID NOT NULL REFERENCES pay_rule(id),
    hourly_rate     NUMERIC(10,4),                      -- base hourly rate
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',     -- ISO 4217
    effective_from  DATE NOT NULL,
    effective_to    DATE,                                -- NULL = current
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, effective_from)
);

CREATE INDEX idx_emp_pay_employee ON employee_pay_assignment (employee_id);

-- ============================================================
-- SHIFT DIFFERENTIAL
-- ============================================================
CREATE TABLE shift_differential (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,              -- e.g., 'Night Shift Premium', 'Weekend Premium'
    differential_type VARCHAR(20) NOT NULL,              -- flat_amount, percentage
    amount          NUMERIC(10,4) NOT NULL,              -- flat $ or percentage multiplier
    start_time      TIME NOT NULL,                       -- e.g., 18:00:00
    end_time        TIME NOT NULL,                       -- e.g., 06:00:00
    applicable_days SMALLINT[] NOT NULL DEFAULT '{0,1,2,3,4,5,6}', -- days of week
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shift_diff_tenant ON shift_differential (tenant_id);

-- ============================================================
-- HOLIDAY
-- ============================================================
CREATE TABLE holiday (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    name            VARCHAR(255) NOT NULL,
    holiday_date    DATE NOT NULL,
    pay_multiplier  NUMERIC(4,2) NOT NULL DEFAULT 1.50, -- holiday premium
    is_paid         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, holiday_date, COALESCE(jurisdiction_id, '00000000-0000-0000-0000-000000000000'))
);

CREATE INDEX idx_holiday_tenant_date ON holiday (tenant_id, holiday_date);
```

## Scheduling

```sql
-- ============================================================
-- SCHEDULE (a named schedule period)
-- ============================================================
CREATE TABLE schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    name            VARCHAR(255) NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, published, archived
    published_at    TIMESTAMPTZ,
    published_by    UUID REFERENCES employee(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_schedule_tenant ON schedule (tenant_id);
CREATE INDEX idx_schedule_location ON schedule (location_id, period_start);

-- ============================================================
-- SHIFT (individual shift assignment)
-- ============================================================
CREATE TABLE shift (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schedule_id     UUID NOT NULL REFERENCES schedule(id) ON DELETE CASCADE,
    employee_id     UUID REFERENCES employee(id),        -- NULL = open/unfilled shift
    location_id     UUID NOT NULL REFERENCES location(id),
    department_id   UUID REFERENCES department(id),
    shift_date      DATE NOT NULL,
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    break_minutes   INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'scheduled', -- scheduled, confirmed, swapped, cancelled
    required_skill_id UUID REFERENCES skill(id),         -- skill/cert required for this shift
    notes           TEXT,
    ical_uid        VARCHAR(255),                        -- RFC 5545 UID for calendar export
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shift_schedule ON shift (schedule_id);
CREATE INDEX idx_shift_employee ON shift (employee_id, shift_date);
CREATE INDEX idx_shift_location_date ON shift (location_id, shift_date);
CREATE INDEX idx_shift_open ON shift (schedule_id) WHERE employee_id IS NULL;

-- ============================================================
-- SHIFT SWAP REQUEST
-- ============================================================
CREATE TABLE shift_swap_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shift_id        UUID NOT NULL REFERENCES shift(id),
    requesting_employee_id UUID NOT NULL REFERENCES employee(id),
    receiving_employee_id UUID REFERENCES employee(id),  -- NULL = open marketplace
    status          VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, accepted, approved, rejected, cancelled
    manager_approval_required BOOLEAN NOT NULL DEFAULT true,
    approved_by     UUID REFERENCES employee(id),
    approved_at     TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_swap_shift ON shift_swap_request (shift_id);
CREATE INDEX idx_swap_requesting ON shift_swap_request (requesting_employee_id);

-- ============================================================
-- EMPLOYEE AVAILABILITY
-- ============================================================
CREATE TABLE employee_availability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    day_of_week     SMALLINT NOT NULL,                   -- 0=Sunday ... 6=Saturday
    available_from  TIME,                                -- NULL = not available
    available_to    TIME,
    is_available    BOOLEAN NOT NULL DEFAULT true,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_availability_employee ON employee_availability (employee_id);
```

## Leave & Absence Management

```sql
-- ============================================================
-- LEAVE TYPE
-- ============================================================
CREATE TABLE leave_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,              -- e.g., 'Vacation', 'Sick', 'FMLA', 'Bereavement'
    code            VARCHAR(20) NOT NULL,
    category        VARCHAR(30) NOT NULL,               -- paid, unpaid, protected, statutory
    is_fmla_qualifying BOOLEAN NOT NULL DEFAULT false,
    is_protected    BOOLEAN NOT NULL DEFAULT false,     -- cannot count toward attendance occurrences
    accrual_method  VARCHAR(20),                        -- per_pay_period, per_hour_worked, annual_grant, none
    accrual_rate    NUMERIC(10,4),                      -- hours per period or per hour worked
    max_balance     NUMERIC(10,2),                      -- maximum accrued balance (hours)
    carryover_limit NUMERIC(10,2),                      -- max hours carried to next year
    requires_approval BOOLEAN NOT NULL DEFAULT true,
    min_notice_days INTEGER DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

-- ============================================================
-- LEAVE BALANCE
-- ============================================================
CREATE TABLE leave_balance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    leave_type_id   UUID NOT NULL REFERENCES leave_type(id),
    balance_hours   NUMERIC(10,2) NOT NULL DEFAULT 0,
    accrued_ytd     NUMERIC(10,2) NOT NULL DEFAULT 0,
    used_ytd        NUMERIC(10,2) NOT NULL DEFAULT 0,
    year            SMALLINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, leave_type_id, year)
);

CREATE INDEX idx_leave_balance_employee ON leave_balance (employee_id);

-- ============================================================
-- LEAVE REQUEST
-- ============================================================
CREATE TABLE leave_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    leave_type_id   UUID NOT NULL REFERENCES leave_type(id),
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    start_half_day  BOOLEAN NOT NULL DEFAULT false,     -- morning/afternoon half-day
    end_half_day    BOOLEAN NOT NULL DEFAULT false,
    total_hours     NUMERIC(10,2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, approved, rejected, cancelled
    reason          TEXT,
    approved_by     UUID REFERENCES employee(id),
    approved_at     TIMESTAMPTZ,
    rejection_reason TEXT,
    fmla_case_id    UUID REFERENCES fmla_case(id),      -- if linked to FMLA case
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_leave_request_employee ON leave_request (employee_id, start_date);
CREATE INDEX idx_leave_request_status ON leave_request (tenant_id, status);

-- ============================================================
-- FMLA CASE (Family Medical Leave Act tracking)
-- ============================================================
CREATE TABLE fmla_case (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    reason_category VARCHAR(50) NOT NULL,               -- serious_health_condition, family_care, military_caregiver, qualifying_exigency
    start_date      DATE NOT NULL,
    end_date        DATE,                                -- calculated: start + 12 weeks
    total_entitlement_hours NUMERIC(10,2) NOT NULL DEFAULT 480, -- 12 weeks * 40 hours
    used_hours      NUMERIC(10,2) NOT NULL DEFAULT 0,
    is_intermittent BOOLEAN NOT NULL DEFAULT false,
    medical_certification_received BOOLEAN NOT NULL DEFAULT false,
    medical_certification_date DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'active', -- active, exhausted, closed
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fmla_employee ON fmla_case (employee_id);
CREATE INDEX idx_fmla_status ON fmla_case (tenant_id, status);

-- ============================================================
-- ATTENDANCE OCCURRENCE (points-based tracking)
-- ============================================================
CREATE TABLE attendance_occurrence (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    occurrence_date DATE NOT NULL,
    type            VARCHAR(30) NOT NULL,               -- tardy, no_call_no_show, early_departure, absence
    points          NUMERIC(4,1) NOT NULL DEFAULT 1.0,
    time_entry_id   UUID REFERENCES time_entry(id),
    notes           TEXT,
    is_excused      BOOLEAN NOT NULL DEFAULT false,
    excused_by      UUID REFERENCES employee(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_occurrence_employee ON attendance_occurrence (employee_id, occurrence_date);
```

## Biometric Data & Consent

```sql
-- ============================================================
-- BIOMETRIC TEMPLATE (stored locally for data sovereignty)
-- ============================================================
CREATE TABLE biometric_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    biometric_type  VARCHAR(30) NOT NULL,               -- facial_embedding, fingerprint_minutiae
    template_data   BYTEA NOT NULL,                      -- encrypted biometric template
    template_standard VARCHAR(30),                       -- ISO_19794_5, ISO_19794_2, proprietary
    algorithm_name  VARCHAR(100) NOT NULL,               -- e.g., 'FaceNet v1.2', 'DeepFace MTCNN'
    algorithm_version VARCHAR(20) NOT NULL,
    quality_score   NUMERIC(5,3),                        -- ISO 19795 quality score
    encryption_key_id VARCHAR(100) NOT NULL,             -- reference to key management system
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    enrolled_by     UUID REFERENCES employee(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    scheduled_destruction_date DATE,                     -- BIPA/GDPR retention limit
    destroyed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_biometric_employee ON biometric_template (employee_id);
CREATE INDEX idx_biometric_destruction ON biometric_template (scheduled_destruction_date) WHERE destroyed_at IS NULL;

-- ============================================================
-- BIOMETRIC CONSENT
-- ============================================================
CREATE TABLE biometric_consent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    consent_type    VARCHAR(30) NOT NULL,               -- facial_recognition, fingerprint
    purpose         TEXT NOT NULL,                       -- "Time and attendance verification"
    retention_period_days INTEGER NOT NULL,              -- e.g., 1095 (3 years)
    consent_given   BOOLEAN NOT NULL,
    consent_text    TEXT NOT NULL,                       -- exact text employee agreed to
    consent_method  VARCHAR(30) NOT NULL,               -- electronic_signature, written
    ip_address      INET,
    user_agent      TEXT,
    given_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ,
    revocation_reason TEXT,
    jurisdiction_law VARCHAR(50),                        -- 'GDPR', 'BIPA', 'CUBI', 'CCPA'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_employee ON biometric_consent (employee_id);
CREATE INDEX idx_consent_active ON biometric_consent (employee_id) WHERE revoked_at IS NULL;
```

## Compliance & Alerts

```sql
-- ============================================================
-- COMPLIANCE RULE
-- ============================================================
CREATE TABLE compliance_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    rule_type       VARCHAR(50) NOT NULL,               -- max_weekly_hours, min_rest_period, break_required, overtime_threshold
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    threshold_value INTEGER,                            -- value in minutes
    comparison      VARCHAR(10),                        -- gte, lte, eq
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning', -- info, warning, violation
    regulation_reference VARCHAR(100),                   -- e.g., 'FLSA 29 CFR 778', 'EU WTD Art. 3'
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_rule_tenant ON compliance_rule (tenant_id);

-- ============================================================
-- COMPLIANCE ALERT
-- ============================================================
CREATE TABLE compliance_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    compliance_rule_id UUID NOT NULL REFERENCES compliance_rule(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    alert_type      VARCHAR(30) NOT NULL,               -- approaching_threshold, violation, anomaly
    severity        VARCHAR(20) NOT NULL,
    message         TEXT NOT NULL,
    context_data    JSONB,                               -- relevant metrics at time of alert
    -- Example: {"current_weekly_minutes": 2340, "threshold_minutes": 2400, "remaining_minutes": 60}
    status          VARCHAR(20) NOT NULL DEFAULT 'open', -- open, acknowledged, resolved, dismissed
    acknowledged_by UUID REFERENCES employee(id),
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_tenant_status ON compliance_alert (tenant_id, status);
CREATE INDEX idx_alert_employee ON compliance_alert (employee_id);
CREATE INDEX idx_alert_created ON compliance_alert (created_at);
```

## Integration & API

```sql
-- ============================================================
-- PAYROLL EXPORT BATCH
-- ============================================================
CREATE TABLE payroll_export (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    export_format   VARCHAR(30) NOT NULL,               -- csv, hr_json_timecard, quickbooks, xero, adp
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, processing, completed, failed
    file_url        VARCHAR(500),
    record_count    INTEGER,
    exported_by     UUID NOT NULL REFERENCES employee(id),
    exported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    error_message   TEXT
);

CREATE INDEX idx_payroll_export_tenant ON payroll_export (tenant_id, exported_at);

-- ============================================================
-- WEBHOOK SUBSCRIPTION
-- ============================================================
CREATE TABLE webhook_subscription (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    url             VARCHAR(500) NOT NULL,
    event_types     TEXT[] NOT NULL,                     -- e.g., '{timesheet.approved, shift.swapped}'
    secret_hash     VARCHAR(255) NOT NULL,               -- HMAC signing secret (hashed)
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_triggered_at TIMESTAMPTZ,
    failure_count   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- OAUTH CLIENT (API consumers)
-- ============================================================
CREATE TABLE oauth_client (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    client_id       VARCHAR(100) NOT NULL UNIQUE,
    client_secret_hash VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    redirect_uris   TEXT[],
    grant_types     TEXT[] NOT NULL DEFAULT '{client_credentials}',
    scopes          TEXT[] NOT NULL DEFAULT '{read}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- NOTIFICATION
-- ============================================================
CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    recipient_id    UUID NOT NULL REFERENCES employee(id),
    type            VARCHAR(50) NOT NULL,               -- shift_reminder, approval_needed, compliance_alert, swap_request
    title           VARCHAR(255) NOT NULL,
    body            TEXT,
    channel         VARCHAR(20) NOT NULL DEFAULT 'in_app', -- in_app, email, push, sms
    reference_type  VARCHAR(30),                        -- shift, timesheet, leave_request, compliance_alert
    reference_id    UUID,
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notification_recipient ON notification (recipient_id, read_at);
CREATE INDEX idx_notification_unread ON notification (recipient_id) WHERE read_at IS NULL;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Organization | 4 | tenant, location, department, jurisdiction |
| Employee & Identity | 7 | employee, employee_location, skill, employee_skill, role, permission, role_permission, employee_role |
| Time Capture | 4 | time_entry, timesheet, timesheet_line, time_entry_audit |
| Pay Rules & Compensation | 4 | pay_rule, employee_pay_assignment, shift_differential, holiday |
| Scheduling | 4 | schedule, shift, shift_swap_request, employee_availability |
| Leave & Absence | 5 | leave_type, leave_balance, leave_request, fmla_case, attendance_occurrence |
| Biometric | 2 | biometric_template, biometric_consent |
| Compliance | 2 | compliance_rule, compliance_alert |
| Integration & API | 4 | payroll_export, webhook_subscription, oauth_client, notification |
| **Total** | **~36** | Core tables; additional lookup/reference tables may be needed |

---

## Key Design Decisions

1. **UUID primary keys everywhere.** Supports distributed deployment, prevents ID enumeration attacks, and simplifies multi-tenant data migration.

2. **Tenant-level multi-tenancy with shared tables.** All core tables include a `tenant_id` column. PostgreSQL RLS policies can enforce row-level isolation. This avoids schema-per-tenant complexity while maintaining data separation.

3. **Separate `time_entry` and `timesheet` tables.** Raw punch events (time_entry) are immutable records of clock actions. Timesheets are aggregated, approvable summaries. This separation mirrors the DCAA requirement for contemporaneous recording distinct from derived reporting.

4. **Jurisdiction-aware pay rules.** The `pay_rule` table is linked to `jurisdiction` (ISO 3166-based), allowing different overtime thresholds and rest period rules per country/state. California double-time, EU WTD 48-hour caps, and FLSA 40-hour weekly overtime all coexist as different pay_rule rows.

5. **FLSA overtime as configuration, not code.** Overtime thresholds, multipliers, and workweek start days are stored in `pay_rule` rows, not hardcoded. This makes the system adaptable to any jurisdiction's overtime rules without code changes.

6. **Explicit biometric consent tracking.** The `biometric_consent` table satisfies Illinois BIPA, GDPR Article 9, and Texas CUBI requirements by recording exactly what the employee consented to, when, how, and under which jurisdiction's law. Revocation is tracked separately from granting.

7. **Scoped RBAC.** The `employee_role` table supports global, location-scoped, and department-scoped role assignments. A manager can approve timesheets for their location without seeing other locations' data.

8. **Immutable time entry audit trail.** The `time_entry_audit` table creates an append-only history of every change to a time entry, satisfying DCAA and FLSA audit requirements. The main `time_entry` record stores the current state; the audit table stores every prior state.

9. **PostGIS-ready geofencing.** Location geofences are stored as GeoJSON in the `geofence_polygon` JSONB column (or `geofence_radius_meters` for circular fences). These can be upgraded to PostGIS `GEOMETRY` columns for spatial queries without schema changes.

10. **iCalendar-compatible shift design.** The `shift` table includes `ical_uid` for RFC 5545 export compatibility, and the `start_time`/`end_time` TIMESTAMPTZ columns map directly to VEVENT DTSTART/DTEND properties.
