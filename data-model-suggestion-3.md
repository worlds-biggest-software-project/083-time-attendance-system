# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Time & Attendance System · Created: 2026-05-12

## Philosophy

This model uses relational tables for core structural data (employees, time entries, schedules) but offloads variable, jurisdiction-specific, and rapidly evolving data into JSONB columns. The approach recognizes that a time and attendance system must serve dozens of jurisdictions (US federal, 50 US states, EU member states, Australian modern awards) each with different pay rules, break requirements, leave categories, and compliance fields. Rather than creating columns or tables for every jurisdiction variant, JSONB columns absorb that variability.

This is the architecture that modern SaaS platforms like Factorial and Deputy use in practice. Core fields that every deployment needs (employee name, punch time, shift start/end) are strongly typed relational columns with indexes and constraints. Jurisdiction-specific fields (California meal break waiver flags, Australian award interpretation codes, German Arbeitszeitgesetz maximum daily hours) live in JSONB columns that can be extended without schema migrations.

The trade-off is reduced type safety for JSONB fields and more complex validation logic (enforced in application code or JSON Schema rather than database constraints). But the benefit is dramatic: adding support for a new jurisdiction's compliance rules is a configuration change, not a database migration. For a product targeting SMBs across multiple countries, this flexibility is essential for rapid iteration.

**Best for:** Multi-jurisdiction deployments, rapid MVP development, and products that must support many country/state-specific compliance rules without per-jurisdiction schema changes.

**Trade-offs:**
- (+) Add new jurisdiction-specific fields without database migrations
- (+) Fewer tables (~25-30) than fully normalized model; simpler to understand
- (+) JSONB indexes (GIN) enable efficient querying of variable fields
- (+) Rapid MVP: core schema is small; extensibility is built in from day one
- (+) Natural fit for multi-tenant SaaS with jurisdiction-per-tenant variation
- (-) JSONB fields lack database-enforced type constraints; validation moves to application layer
- (-) JSON Schema validation must be maintained alongside the database schema
- (-) Complex JSONB queries can be slower than indexed relational columns for analytics
- (-) Refactoring a JSONB field into a relational column requires data migration
- (-) Harder to enforce referential integrity within JSONB structures

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FLSA (29 CFR Part 778) | Base overtime rules in relational `pay_rule` columns; state-specific overrides (CA double-time, 7th-day OT) in `pay_rule.jurisdiction_overrides` JSONB |
| FMLA | Core leave tracking relational; intermittent leave specifics and state-level FMLA extensions in `leave_request.metadata` JSONB |
| EU Working Time Directive | WTD rules stored in `compliance_config.rules` JSONB, allowing per-member-state variations without separate tables |
| DCAA Timekeeping | Project/labor category allocation in `time_entry.work_allocation` JSONB for flexible cost structure support |
| ISO 3166 | Relational `country_code` and `subdivision_code` columns on location and jurisdiction tables |
| GDPR / BIPA | Biometric consent details in `biometric_consent.consent_details` JSONB capturing jurisdiction-specific consent requirements |
| HR Open Standards (HR-JSON Timecard) | Payroll export maps relational fields + JSONB extensions to HR-JSON Timecard schema; JSONB structure follows HR-JSON field naming conventions where possible |
| RFC 5545 (iCalendar) | Shift `ical_properties` JSONB stores RRULE, EXDATE, and custom iCalendar properties for complex recurring schedules |
| ISO/IEC 19794-5 | Biometric metadata in JSONB; template standard and algorithm captured as structured JSON |
| OpenAPI 3.x / JSON Schema | JSONB columns validated against JSON Schema documents; schemas published alongside the OpenAPI spec |

---

## Core Tables

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
    subscription_plan VARCHAR(50) NOT NULL DEFAULT 'free',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "workweek_start": 1,
    --   "default_break_minutes": 30,
    --   "require_biometric": false,
    --   "geofencing_enabled": true,
    --   "approval_workflow": "single_manager",
    --   "payroll_export_format": "csv",
    --   "branding": { "logo_url": "...", "primary_color": "#1a73e8" },
    --   "enabled_modules": ["scheduling", "leave", "compliance"],
    --   "dcaa_mode": false
    -- }
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
    country_code    CHAR(2) NOT NULL,                   -- ISO 3166-1
    subdivision_code VARCHAR(6),                         -- ISO 3166-2
    timezone        VARCHAR(50) NOT NULL,
    address         JSONB NOT NULL DEFAULT '{}',
    -- address example:
    -- {
    --   "line1": "123 Main St",
    --   "line2": "Suite 400",
    --   "city": "Chicago",
    --   "state": "IL",
    --   "postal_code": "60601",
    --   "formatted": "123 Main St, Suite 400, Chicago, IL 60601"
    -- }
    geofence        JSONB NOT NULL DEFAULT '{}',
    -- geofence example (circular):
    -- { "type": "circle", "center": {"lat": 41.878, "lng": -87.630}, "radius_meters": 150 }
    -- geofence example (polygon / GeoJSON):
    -- { "type": "Polygon", "coordinates": [[[lng,lat], [lng,lat], ...]] }
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
    parent_id       UUID REFERENCES department(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    cost_center     VARCHAR(50),
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- { "budget_code": "ENG-2026", "head_count_target": 45 }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_department_tenant ON department (tenant_id);
CREATE INDEX idx_department_parent ON department (parent_id);
```

## Employee

```sql
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
    primary_location_id UUID REFERENCES location(id),
    primary_department_id UUID REFERENCES department(id),
    manager_id      UUID REFERENCES employee(id),
    timezone        VARCHAR(50),
    hourly_rate     NUMERIC(10,4),
    currency_code   CHAR(3) DEFAULT 'USD',
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "date_of_birth": "1990-03-15",
    --   "emergency_contact": { "name": "Jane Doe", "phone": "+1-555-1234" },
    --   "tax_id_last4": "5678",
    --   "shirt_size": "L",
    --   "preferred_name": "Mike",
    --   "languages": ["en", "es"],
    --   "certifications": [
    --     { "name": "Certified Welder", "number": "CW-12345", "expiry": "2027-06-30" }
    --   ]
    -- }
    pay_config      JSONB NOT NULL DEFAULT '{}',
    -- pay_config example:
    -- {
    --   "pay_frequency": "biweekly",
    --   "shift_differentials": [
    --     { "name": "Night Premium", "type": "flat", "amount": 2.50,
    --       "start_time": "18:00", "end_time": "06:00" }
    --   ],
    --   "custom_pay_codes": ["hazard_pay", "training_pay"]
    -- }
    compliance_data JSONB NOT NULL DEFAULT '{}',
    -- compliance_data example:
    -- {
    --   "jurisdiction": "US-CA",
    --   "overtime_rule": "california",
    --   "meal_break_waiver_signed": true,
    --   "i9_verified": true,
    --   "i9_verified_date": "2024-01-15",
    --   "dcaa_labor_categories": ["senior_engineer", "project_lead"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, employee_number)
);

CREATE INDEX idx_employee_tenant ON employee (tenant_id);
CREATE INDEX idx_employee_status ON employee (tenant_id, employment_status);
CREATE INDEX idx_employee_manager ON employee (manager_id);
CREATE INDEX idx_employee_profile ON employee USING GIN (profile);
CREATE INDEX idx_employee_compliance ON employee USING GIN (compliance_data);
```

## Time Capture

```sql
-- ============================================================
-- TIME ENTRY (punch / clock event)
-- ============================================================
CREATE TABLE time_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    entry_type      VARCHAR(20) NOT NULL,               -- clock_in, clock_out, break_start, break_end
    punch_time      TIMESTAMPTZ NOT NULL,
    adjusted_time   TIMESTAMPTZ,
    source          VARCHAR(30) NOT NULL,               -- web, mobile_app, kiosk, manual, api
    location_id     UUID REFERENCES location(id),
    geolocation     JSONB,
    -- geolocation example:
    -- {
    --   "latitude": 41.8781136,
    --   "longitude": -87.6297982,
    --   "accuracy_meters": 5.2,
    --   "geofence_status": "inside",
    --   "geofence_distance_meters": 12,
    --   "altitude": 182.3
    -- }
    biometric       JSONB,
    -- biometric example:
    -- {
    --   "method": "facial_recognition",
    --   "confidence": 0.987,
    --   "algorithm": "FaceNet v1.2",
    --   "liveness_check": true,
    --   "template_id": "uuid"
    -- }
    work_allocation JSONB,
    -- work_allocation example (for DCAA / project tracking):
    -- {
    --   "project_code": "PROJ-2026-042",
    --   "labor_category": "senior_engineer",
    --   "task_code": "design_review",
    --   "contract_number": "FA8750-26-C-0001",
    --   "cost_center": "ENG-CORE"
    -- }
    flags           JSONB NOT NULL DEFAULT '[]',
    -- flags example:
    -- [
    --   { "type": "outside_geofence", "detected_at": "...", "resolved": false },
    --   { "type": "low_biometric_confidence", "confidence": 0.62, "resolved": true,
    --     "resolved_by": "uuid", "resolved_at": "..." }
    -- ]
    device_info     JSONB,
    -- device_info example:
    -- { "device_id": "iPhone14,2-ABCD", "os": "iOS 19.1", "app_version": "2.4.1",
    --   "ip_address": "192.168.1.42", "user_agent": "..." }
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_te_employee_date ON time_entry (employee_id, punch_time);
CREATE INDEX idx_te_tenant_date ON time_entry (tenant_id, punch_time);
CREATE INDEX idx_te_flags ON time_entry USING GIN (flags) WHERE flags != '[]'::jsonb;

-- ============================================================
-- TIME ENTRY AUDIT (change history)
-- ============================================================
CREATE TABLE time_entry_audit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    time_entry_id   UUID NOT NULL REFERENCES time_entry(id),
    changes         JSONB NOT NULL,
    -- changes example:
    -- {
    --   "adjusted_time": { "old": null, "new": "2026-05-12T08:00:00-05:00" },
    --   "flags": { "old": [...], "new": [...] }
    -- }
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
    hours_summary   JSONB NOT NULL DEFAULT '{}',
    -- hours_summary example:
    -- {
    --   "regular_minutes": 2400,
    --   "overtime_minutes": 120,
    --   "doubletime_minutes": 0,
    --   "break_minutes": 150,
    --   "pto_minutes": 0,
    --   "holiday_minutes": 0,
    --   "total_worked_minutes": 2520,
    --   "by_day": {
    --     "2026-05-04": { "regular": 480, "overtime": 0, "break": 30 },
    --     "2026-05-05": { "regular": 480, "overtime": 60, "break": 30 },
    --     ...
    --   },
    --   "by_project": {
    --     "PROJ-2026-042": { "minutes": 1200 },
    --     "PROJ-2026-043": { "minutes": 1320 }
    --   },
    --   "by_location": {
    --     "uuid-location-1": { "minutes": 2520 }
    --   }
    -- }
    approval        JSONB NOT NULL DEFAULT '{}',
    -- approval example:
    -- {
    --   "submitted_at": "...",
    --   "submitted_by": "uuid",
    --   "approvals": [
    --     { "level": 1, "approved_by": "uuid", "approved_at": "...", "notes": "Looks good" }
    --   ],
    --   "rejection": null
    -- }
    pay_calculations JSONB NOT NULL DEFAULT '{}',
    -- pay_calculations example:
    -- {
    --   "regular_rate": 25.00,
    --   "overtime_rate": 37.50,
    --   "doubletime_rate": 50.00,
    --   "shift_differentials": [
    --     { "name": "Night Premium", "hours": 16, "amount_per_hour": 2.50, "total": 40.00 }
    --   ],
    --   "gross_pay_estimate": 1037.50,
    --   "currency": "USD"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, period_start, period_type)
);

CREATE INDEX idx_timesheet_employee ON timesheet (employee_id, period_start);
CREATE INDEX idx_timesheet_status ON timesheet (tenant_id, status);
```

## Pay Rules & Compliance

```sql
-- ============================================================
-- PAY RULE (with jurisdiction overrides in JSONB)
-- ============================================================
CREATE TABLE pay_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    -- Core overtime fields (relational for common queries)
    workweek_start_day SMALLINT NOT NULL DEFAULT 0,
    overtime_weekly_threshold INTEGER NOT NULL DEFAULT 2400,
    overtime_multiplier NUMERIC(4,2) NOT NULL DEFAULT 1.50,
    -- Jurisdiction-specific overrides and extensions
    rules           JSONB NOT NULL DEFAULT '{}',
    -- rules example (California):
    -- {
    --   "overtime_daily_threshold": 480,
    --   "doubletime_daily_threshold": 720,
    --   "doubletime_multiplier": 2.00,
    --   "seventh_day_overtime": true,
    --   "seventh_day_doubletime_after": 480,
    --   "meal_break": { "after_minutes": 300, "duration_minutes": 30, "paid": false, "waivable": true },
    --   "rest_break": { "every_minutes": 240, "duration_minutes": 10, "paid": true },
    --   "split_shift_premium": true
    -- }
    -- rules example (EU - Germany):
    -- {
    --   "max_daily_hours": 600,
    --   "max_weekly_hours": 2880,
    --   "min_daily_rest_minutes": 660,
    --   "min_weekly_rest_minutes": 1440,
    --   "sunday_work_restricted": true,
    --   "night_work_max_hours": 480,
    --   "break_rules": [
    --     { "after_minutes": 360, "min_break_minutes": 30 },
    --     { "after_minutes": 540, "min_break_minutes": 45 }
    --   ]
    -- }
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pay_rule_tenant ON pay_rule (tenant_id);
CREATE INDEX idx_pay_rule_country ON pay_rule (country_code, subdivision_code);

-- ============================================================
-- COMPLIANCE CONFIG (per-tenant compliance settings)
-- ============================================================
CREATE TABLE compliance_config (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    rules           JSONB NOT NULL DEFAULT '[]',
    -- rules example:
    -- [
    --   {
    --     "rule_type": "approaching_overtime",
    --     "regulation": "FLSA",
    --     "threshold_minutes": 2280,
    --     "severity": "warning",
    --     "notification_channels": ["in_app", "email"],
    --     "notify_roles": ["manager"]
    --   },
    --   {
    --     "rule_type": "max_weekly_hours",
    --     "regulation": "EU WTD Art. 6",
    --     "threshold_minutes": 2880,
    --     "severity": "violation",
    --     "auto_block_scheduling": true
    --   },
    --   {
    --     "rule_type": "missed_break",
    --     "regulation": "California Labor Code §512",
    --     "after_minutes": 300,
    --     "required_break_minutes": 30,
    --     "penalty_type": "one_hour_pay"
    --   }
    -- ]
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_config_tenant ON compliance_config (tenant_id);

-- ============================================================
-- COMPLIANCE ALERT
-- ============================================================
CREATE TABLE compliance_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    alert_data      JSONB NOT NULL,
    -- alert_data example:
    -- {
    --   "rule_type": "approaching_overtime",
    --   "regulation": "FLSA",
    --   "severity": "warning",
    --   "current_weekly_minutes": 2340,
    --   "threshold_minutes": 2400,
    --   "remaining_minutes": 60,
    --   "week_start": "2026-05-04",
    --   "triggered_by_time_entry_id": "uuid"
    -- }
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    acknowledged_by UUID REFERENCES employee(id),
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_tenant_status ON compliance_alert (tenant_id, status);
CREATE INDEX idx_alert_employee ON compliance_alert (employee_id);
CREATE INDEX idx_alert_data ON compliance_alert USING GIN (alert_data);
```

## Scheduling

```sql
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
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "auto_schedule_enabled": true,
    --   "demand_source": "pos_integration",
    --   "min_coverage": { "morning": 3, "afternoon": 4, "evening": 2 },
    --   "max_consecutive_days": 6,
    --   "prefer_employee_preferences": true
    -- }
    published_at    TIMESTAMPTZ,
    published_by    UUID REFERENCES employee(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_schedule_tenant ON schedule (tenant_id);
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
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "required_skills": ["certified_welder", "forklift_operator"],
    --   "color": "#4CAF50",
    --   "notes": "Training shift — shadow senior operator",
    --   "shift_type": "training",
    --   "premium_applicable": "night_shift",
    --   "ical_uid": "shift-uuid@example.com",
    --   "ical_properties": {
    --     "RRULE": null,
    --     "CATEGORIES": ["NIGHT_SHIFT"],
    --     "DESCRIPTION": "Night shift at Main Warehouse"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shift_schedule ON shift (schedule_id);
CREATE INDEX idx_shift_employee ON shift (employee_id, shift_date);
CREATE INDEX idx_shift_location ON shift (location_id, shift_date);
CREATE INDEX idx_shift_open ON shift (schedule_id) WHERE employee_id IS NULL;

-- ============================================================
-- SHIFT SWAP
-- ============================================================
CREATE TABLE shift_swap (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shift_id        UUID NOT NULL REFERENCES shift(id),
    from_employee_id UUID NOT NULL REFERENCES employee(id),
    to_employee_id  UUID REFERENCES employee(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "reason": "Doctor appointment",
    --   "manager_approval_required": true,
    --   "approved_by": "uuid",
    --   "approved_at": "...",
    --   "auto_approved": false,
    --   "swap_offer_expires_at": "2026-05-10T18:00:00Z"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_swap_shift ON shift_swap (shift_id);

-- ============================================================
-- EMPLOYEE AVAILABILITY
-- ============================================================
CREATE TABLE employee_availability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    availability    JSONB NOT NULL,
    -- availability example:
    -- {
    --   "monday":    { "available": true, "from": "06:00", "to": "22:00" },
    --   "tuesday":   { "available": true, "from": "06:00", "to": "22:00" },
    --   "wednesday": { "available": true, "from": "06:00", "to": "22:00" },
    --   "thursday":  { "available": true, "from": "06:00", "to": "14:00" },
    --   "friday":    { "available": true, "from": "06:00", "to": "22:00" },
    --   "saturday":  { "available": false },
    --   "sunday":    { "available": false },
    --   "preferences": {
    --     "max_hours_per_week": 40,
    --     "prefer_morning": true,
    --     "avoid_consecutive_nights": true
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_avail_employee ON employee_availability (employee_id);
```

## Leave & Absence

```sql
-- ============================================================
-- LEAVE TYPE
-- ============================================================
CREATE TABLE leave_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    category        VARCHAR(30) NOT NULL,
    is_protected    BOOLEAN NOT NULL DEFAULT false,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "is_fmla_qualifying": true,
    --   "accrual_method": "per_hour_worked",
    --   "accrual_rate": 0.0462,
    --   "max_balance_hours": 120,
    --   "carryover_limit_hours": 40,
    --   "requires_approval": true,
    --   "min_notice_days": 5,
    --   "requires_documentation": false,
    --   "jurisdiction_specific": {
    --     "US-CA": { "sick_leave_rate": 0.0333, "max_balance": 48 },
    --     "US-NY": { "sick_leave_rate": 0.0167, "max_balance": 56 }
    --   }
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

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
    total_hours     NUMERIC(10,2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {
    --   "reason": "Family vacation",
    --   "half_day_start": false,
    --   "half_day_end": true,
    --   "fmla_case_id": "uuid",
    --   "intermittent_schedule": {
    --     "frequency": "2-3 times per month",
    --     "duration_per_episode_hours": 4
    --   },
    --   "approval_chain": [
    --     { "level": 1, "approver_id": "uuid", "status": "approved", "at": "..." }
    --   ],
    --   "supporting_documents": [
    --     { "name": "medical_cert.pdf", "url": "/docs/...", "uploaded_at": "..." }
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_leave_employee ON leave_request (employee_id, start_date);
CREATE INDEX idx_leave_status ON leave_request (tenant_id, status);

-- ============================================================
-- LEAVE BALANCE
-- ============================================================
CREATE TABLE leave_balance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    leave_type_id   UUID NOT NULL REFERENCES leave_type(id),
    year            SMALLINT NOT NULL,
    balance_hours   NUMERIC(10,2) NOT NULL DEFAULT 0,
    accrued_ytd     NUMERIC(10,2) NOT NULL DEFAULT 0,
    used_ytd        NUMERIC(10,2) NOT NULL DEFAULT 0,
    adjustments     JSONB NOT NULL DEFAULT '[]',
    -- adjustments example:
    -- [
    --   { "date": "2026-01-01", "hours": 40, "reason": "Annual carryover", "by": "system" },
    --   { "date": "2026-03-15", "hours": -8, "reason": "Correction", "by": "uuid" }
    -- ]
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, leave_type_id, year)
);
```

## Biometric & Security

```sql
-- ============================================================
-- BIOMETRIC TEMPLATE
-- ============================================================
CREATE TABLE biometric_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    biometric_type  VARCHAR(30) NOT NULL,
    template_data   BYTEA NOT NULL,                     -- encrypted
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {
    --   "algorithm": "FaceNet v1.2",
    --   "algorithm_version": "1.2.0",
    --   "template_standard": "ISO_19794_5",
    --   "quality_score": 0.945,
    --   "enrollment_device": "iPad Pro 12.9 (6th gen)",
    --   "encryption_key_id": "key-2026-05"
    -- }
    scheduled_destruction_date DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_biometric_employee ON biometric_template (employee_id);

-- ============================================================
-- BIOMETRIC CONSENT
-- ============================================================
CREATE TABLE biometric_consent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    consent_type    VARCHAR(30) NOT NULL,
    consent_given   BOOLEAN NOT NULL,
    consent_details JSONB NOT NULL,
    -- consent_details example:
    -- {
    --   "purpose": "Time and attendance verification via facial recognition",
    --   "retention_period_days": 1095,
    --   "consent_text": "I consent to the collection...",
    --   "consent_method": "electronic_signature",
    --   "jurisdiction_law": "BIPA",
    --   "ip_address": "192.168.1.42",
    --   "user_agent": "Mozilla/5.0...",
    --   "witness": null
    -- }
    given_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_employee ON biometric_consent (employee_id);

-- ============================================================
-- RBAC (simplified with JSONB permissions)
-- ============================================================
CREATE TABLE app_role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions example:
    -- [
    --   { "resource": "timesheet", "actions": ["read", "approve"], "scope": "department" },
    --   { "resource": "schedule", "actions": ["read", "write", "publish"], "scope": "location" },
    --   { "resource": "employee", "actions": ["read"], "scope": "department" },
    --   { "resource": "compliance_alert", "actions": ["read", "acknowledge"], "scope": "location" }
    -- ]
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE employee_role (
    employee_id     UUID NOT NULL REFERENCES employee(id),
    role_id         UUID NOT NULL REFERENCES app_role(id),
    scope_id        UUID,                               -- location_id or department_id
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (employee_id, role_id, COALESCE(scope_id, '00000000-0000-0000-0000-000000000000'))
);
```

## Integration

```sql
-- ============================================================
-- PAYROLL EXPORT
-- ============================================================
CREATE TABLE payroll_export (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    export_format   VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    export_data     JSONB,
    -- export_data example:
    -- {
    --   "format_version": "1.0",
    --   "records": [
    --     {
    --       "employee_number": "EMP-001",
    --       "regular_hours": 40.00,
    --       "overtime_hours": 2.00,
    --       "pto_hours": 0,
    --       "pay_codes": [
    --         { "code": "REG", "hours": 40.00, "rate": 25.00 },
    --         { "code": "OT", "hours": 2.00, "rate": 37.50 },
    --         { "code": "NIGHT", "hours": 16.00, "rate": 2.50 }
    --       ]
    --     }
    --   ],
    --   "totals": { "employees": 45, "regular_hours": 1800, "overtime_hours": 62 }
    -- }
    file_url        VARCHAR(500),
    exported_by     UUID NOT NULL REFERENCES employee(id),
    exported_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payroll_export_tenant ON payroll_export (tenant_id, exported_at);

-- ============================================================
-- NOTIFICATION
-- ============================================================
CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    recipient_id    UUID NOT NULL REFERENCES employee(id),
    type            VARCHAR(50) NOT NULL,
    content         JSONB NOT NULL,
    -- content example:
    -- {
    --   "title": "Timesheet Approval Needed",
    --   "body": "John Smith submitted their timesheet for May 4-10",
    --   "reference_type": "timesheet",
    --   "reference_id": "uuid",
    --   "action_url": "/timesheets/uuid",
    --   "priority": "normal"
    -- }
    channel         VARCHAR(20) NOT NULL DEFAULT 'in_app',
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notification_recipient ON notification (recipient_id, read_at);
CREATE INDEX idx_notification_unread ON notification (recipient_id) WHERE read_at IS NULL;
```

## Example JSONB Queries

```sql
-- ============================================================
-- Find all employees with California overtime rules
-- ============================================================
SELECT id, first_name, last_name, compliance_data->>'jurisdiction' AS jurisdiction
FROM employee
WHERE tenant_id = '{{tenant_uuid}}'
  AND compliance_data->>'jurisdiction' = 'US-CA';

-- ============================================================
-- Find employees with expiring certifications
-- ============================================================
SELECT
    e.id,
    e.first_name,
    e.last_name,
    cert->>'name' AS certification,
    cert->>'expiry' AS expiry_date
FROM employee e,
     jsonb_array_elements(e.profile->'certifications') AS cert
WHERE e.tenant_id = '{{tenant_uuid}}'
  AND (cert->>'expiry')::date <= now() + INTERVAL '90 days'
  AND (cert->>'expiry')::date >= now();

-- ============================================================
-- Find all flagged time entries with unresolved geofence issues
-- ============================================================
SELECT
    te.id,
    te.employee_id,
    te.punch_time,
    flag->>'type' AS flag_type,
    te.geolocation->>'geofence_distance_meters' AS distance
FROM time_entry te,
     jsonb_array_elements(te.flags) AS flag
WHERE te.tenant_id = '{{tenant_uuid}}'
  AND flag->>'type' = 'outside_geofence'
  AND (flag->>'resolved')::boolean = false;

-- ============================================================
-- Aggregate overtime by jurisdiction for compliance reporting
-- ============================================================
SELECT
    e.compliance_data->>'jurisdiction' AS jurisdiction,
    COUNT(DISTINCT e.id) AS employee_count,
    SUM((ts.hours_summary->>'overtime_minutes')::integer) AS total_overtime_minutes
FROM timesheet ts
JOIN employee e ON ts.employee_id = e.id
WHERE ts.tenant_id = '{{tenant_uuid}}'
  AND ts.period_start >= '2026-05-01'
  AND ts.period_end <= '2026-05-31'
GROUP BY e.compliance_data->>'jurisdiction'
ORDER BY total_overtime_minutes DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Organization | 3 | tenant, location, department |
| Employee | 1 | Single table with profile/pay_config/compliance_data JSONB |
| Time Capture | 3 | time_entry, time_entry_audit, timesheet |
| Pay Rules & Compliance | 3 | pay_rule, compliance_config, compliance_alert |
| Scheduling | 4 | schedule, shift, shift_swap, employee_availability |
| Leave & Absence | 3 | leave_type, leave_request, leave_balance |
| Biometric & Security | 4 | biometric_template, biometric_consent, app_role, employee_role |
| Integration | 2 | payroll_export, notification |
| **Total** | **~23** | Significantly fewer tables than normalized model; extensibility via JSONB |

---

## Key Design Decisions

1. **Relational core, JSONB extensions.** Columns queried in WHERE clauses and JOIN conditions (employee_id, tenant_id, punch_time, status) are relational. Variable, jurisdiction-specific, or rarely-queried data lives in JSONB. This minimizes table count while preserving query performance for hot paths.

2. **GIN indexes on JSONB columns.** Critical JSONB columns (employee.profile, employee.compliance_data, time_entry.flags, compliance_alert.alert_data) have GIN indexes for efficient containment queries (`@>` operator).

3. **Tenant config as JSONB.** Tenant-level settings (enabled modules, default break duration, approval workflow type, branding) are a single JSONB column rather than a wide table with 50+ columns, most of which would be NULL for any given tenant.

4. **Pay rules with jurisdiction overrides.** Core overtime fields (weekly threshold, multiplier) are relational for fast queries. Jurisdiction-specific extensions (California daily OT, EU WTD rest periods, German Sunday restrictions) live in `pay_rule.rules` JSONB. Adding a new jurisdiction requires only inserting a new row, not altering the schema.

5. **Timesheet hours_summary as JSONB.** The timesheet aggregates hours by day, by project, and by location in a single JSONB column. This avoids the need for separate timesheet_line and timesheet_project tables, reducing JOIN complexity for the timesheet approval UI.

6. **Employee availability as JSONB.** Weekly availability patterns with preferences are naturally structured data that varies significantly between employees. A single JSONB column replaces what would be 7+ rows in a normalized availability table.

7. **Flags as JSONB array on time_entry.** Anomaly flags (geofence violations, low biometric confidence, buddy-punch suspicion) are stored as a JSONB array directly on the time entry. This co-locates flags with the data they describe and avoids a separate flags table.

8. **Approval workflows in JSONB.** Multi-level approval chains on timesheets and leave requests are stored as JSONB arrays within the parent record. This supports variable approval depth (1 level for SMB, 3 levels for enterprise) without schema changes.

9. **RBAC permissions as JSONB.** Role permissions are defined as a JSONB array of resource/action/scope objects on the role record. This allows fine-grained permission definitions without a three-table join (role -> role_permission -> permission).

10. **JSON Schema validation at application layer.** Each JSONB column has a corresponding JSON Schema definition maintained in the application codebase. Schema validation runs on write operations; the database does not enforce JSONB structure. This is the explicit trade-off for schema flexibility.
