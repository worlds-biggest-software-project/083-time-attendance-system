# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Time & Attendance System · Created: 2026-05-12

## Philosophy

This model treats every state change as an immutable domain event appended to an event store. The event store is the single source of truth; all queryable state (timesheets, schedules, balances, compliance status) is derived by projecting events into materialized read models. The system follows the CQRS (Command Query Responsibility Segregation) pattern: write operations append events, read operations query projections.

This architecture is a natural fit for time and attendance because the domain is fundamentally event-driven. Clock-ins, clock-outs, break starts, shift assignments, approvals, and compliance alerts are all discrete events with a timestamp and an actor. Regulatory frameworks like DCAA and FLSA demand a complete, tamper-evident audit trail of every time record and every change to a time record. Event sourcing provides this by design rather than as an afterthought.

The trade-off is complexity. Event replay, projection management, eventual consistency between the event store and read models, and the need for careful event schema versioning all add engineering overhead. But for a T&A system where "what happened and when" is not just a feature but a legal requirement, the audit guarantees of event sourcing justify the additional complexity.

**Best for:** Deployments where full temporal auditability, DCAA compliance, point-in-time reconstruction ("what was the timesheet state on March 15?"), and AI-powered change pattern analytics are top priorities.

**Trade-offs:**
- (+) Complete, immutable, tamper-evident audit trail by design
- (+) Point-in-time state reconstruction: replay events to see state at any past moment
- (+) Natural fit for compliance: DCAA, FLSA, and GDPR right-to-erasure via crypto-shredding
- (+) AI analytics on event streams: detect anomalies, predict patterns from raw event data
- (+) Independent scaling of writes (event append) and reads (projection queries)
- (-) Higher engineering complexity: event versioning, projection rebuilds, eventual consistency
- (-) Read model staleness: projections may lag behind the event store by milliseconds to seconds
- (-) Debugging requires understanding event replay, not just querying a table
- (-) Storage growth: event store grows indefinitely; snapshots needed for performance
- (-) Team learning curve: not all developers are familiar with event sourcing patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FLSA (29 CFR Part 778) | Overtime calculations derived by projecting `WorkedHoursCalculated` events; every change to time entries is recorded as an immutable event, creating the audit trail FLSA enforcement requires |
| DCAA Timekeeping | Event store is inherently contemporaneous: each `TimeEntryRecorded` event captures the exact moment of recording with actor identity. Supervisor approvals are `TimesheetApproved` events. The full event stream satisfies DCAA's requirement for immutable, chronological records |
| FMLA | Leave events (`LeaveRequested`, `LeaveApproved`, `FMLACaseOpened`) form a complete leave history. Intermittent FMLA usage is tracked as a stream of events, enabling accurate balance calculation at any point in time |
| EU Working Time Directive | Compliance violations detected by projecting events into working hours aggregates. `ComplianceViolationDetected` events are themselves immutable records of when violations were flagged |
| GDPR Article 17 (Right to Erasure) | Implemented via crypto-shredding: employee-specific encryption keys are destroyed, rendering events unreadable while preserving event store integrity and non-personal aggregate data |
| ISO/IEC 19794-5 | `BiometricTemplateEnrolled` events record template metadata; actual biometric data stored in a separate encrypted store referenced by event payload |
| Illinois BIPA | Consent events (`BiometricConsentGranted`, `BiometricConsentRevoked`) create an immutable consent timeline satisfying BIPA's record-keeping requirements |
| HR Open Standards (HR-JSON Timecard) | Payroll export projections transform event data into HR-JSON Timecard format; the projection can be rebuilt or modified without changing the source events |
| RFC 5545 (iCalendar) | Schedule projections generate iCalendar-compatible data structures from `ShiftAssigned`, `ShiftSwapped`, and `ShiftCancelled` events |

---

## Event Store (Write Side)

```sql
-- ============================================================
-- CORE EVENT STORE
-- ============================================================
-- This is the single source of truth. All other tables are
-- derived projections that can be rebuilt by replaying events.
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                      -- aggregate root ID (employee, timesheet, schedule, etc.)
    stream_type     VARCHAR(50) NOT NULL,               -- Employee, Timesheet, Schedule, LeaveRequest, FMLACase
    tenant_id       UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,              -- e.g., 'TimeEntryRecorded', 'TimesheetApproved'
    event_version   INTEGER NOT NULL,                   -- sequential per stream (optimistic concurrency)
    payload         JSONB NOT NULL,                     -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',        -- actor, source, correlation_id, causation_id
    -- metadata example: {"actor_id": "uuid", "source": "mobile_app", "ip": "1.2.3.4",
    --                     "correlation_id": "uuid", "causation_id": "uuid", "device_id": "..."}
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(), -- wall clock time of recording
    event_timestamp TIMESTAMPTZ NOT NULL,               -- domain time (when the action occurred)
    schema_version  SMALLINT NOT NULL DEFAULT 1         -- for event payload versioning/upcasting
);

-- Optimistic concurrency: unique per stream + version
CREATE UNIQUE INDEX idx_event_stream_version ON event_store (stream_id, event_version);

-- Temporal queries: find all events for a stream ordered by version
CREATE INDEX idx_event_stream_ordered ON event_store (stream_id, event_version);

-- Global ordering for projections (replay all events in order)
CREATE INDEX idx_event_recorded ON event_store (recorded_at);

-- Event type filtering for specific projections
CREATE INDEX idx_event_type ON event_store (event_type, recorded_at);

-- Tenant isolation
CREATE INDEX idx_event_tenant ON event_store (tenant_id, recorded_at);

-- Partition by month for storage management (optional, recommended for scale)
-- In production, this table would be partitioned:
-- CREATE TABLE event_store (...) PARTITION BY RANGE (recorded_at);
-- CREATE TABLE event_store_2026_01 PARTITION OF event_store FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- ============================================================
-- EVENT STORE SNAPSHOT (performance optimization)
-- ============================================================
-- Snapshots store the materialized state of an aggregate at a
-- given event version, avoiding full replay from event 0.
-- ============================================================

CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,                   -- event_version at which snapshot was taken
    state           JSONB NOT NULL,                     -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- ============================================================
-- IDEMPOTENCY GUARD (prevent duplicate event processing)
-- ============================================================
CREATE TABLE processed_event (
    projection_name VARCHAR(100) NOT NULL,
    event_id        UUID NOT NULL,
    processed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (projection_name, event_id)
);
```

## Event Type Catalogue

```
-- ============================================================
-- TIME CAPTURE EVENTS
-- ============================================================
-- TimeEntryRecorded          — employee clocked in/out/break
-- TimeEntryAdjusted          — manager corrected a punch time
-- TimeEntryFlagged           — anomaly detected (geofence, biometric)
-- TimeEntryFlagResolved      — flag resolved by manager
-- TimeEntryDeleted           — soft-delete (event records the deletion)

-- ============================================================
-- TIMESHEET EVENTS
-- ============================================================
-- TimesheetCreated           — new timesheet period opened
-- TimesheetLineUpdated       — daily hours recalculated
-- TimesheetSubmitted         — employee submitted for approval
-- TimesheetApproved          — manager approved
-- TimesheetRejected          — manager rejected with reason
-- TimesheetLocked            — locked for payroll export
-- TimesheetUnlocked          — reopened for corrections

-- ============================================================
-- SCHEDULE EVENTS
-- ============================================================
-- ScheduleCreated            — new schedule period created
-- ShiftAssigned              — shift assigned to employee
-- ShiftUnassigned            — shift removed from employee (open shift)
-- ShiftSwapRequested         — employee requested swap
-- ShiftSwapAccepted          — other employee accepted swap
-- ShiftSwapApproved          — manager approved swap
-- ShiftSwapRejected          — manager rejected swap
-- SchedulePublished          — schedule made visible to employees

-- ============================================================
-- LEAVE EVENTS
-- ============================================================
-- LeaveRequested             — employee requested leave
-- LeaveApproved              — manager approved leave
-- LeaveRejected              — manager rejected leave
-- LeaveCancelled             — leave request cancelled
-- LeaveBalanceAdjusted       — manual balance adjustment
-- LeaveAccrued               — automatic accrual event
-- FMLACaseOpened             — new FMLA case
-- FMLACaseClosed             — FMLA case closed/exhausted

-- ============================================================
-- COMPLIANCE EVENTS
-- ============================================================
-- ComplianceThresholdApproaching — nearing OT/hours limit
-- ComplianceViolationDetected    — actual violation occurred
-- ComplianceAlertAcknowledged    — manager acknowledged alert
-- ComplianceAlertResolved        — corrective action taken

-- ============================================================
-- EMPLOYEE LIFECYCLE EVENTS
-- ============================================================
-- EmployeeOnboarded          — new employee added
-- EmployeeUpdated            — profile data changed
-- EmployeeLocationAssigned   — assigned to location
-- EmployeePayRuleAssigned    — pay rule attached
-- EmployeeTerminated         — employment ended
-- EmployeeRoleGranted        — RBAC role assigned
-- EmployeeRoleRevoked        — RBAC role removed

-- ============================================================
-- BIOMETRIC EVENTS
-- ============================================================
-- BiometricConsentGranted    — employee gave biometric consent
-- BiometricConsentRevoked    — employee revoked consent
-- BiometricTemplateEnrolled  — biometric template registered
-- BiometricTemplateDestroyed — template destroyed per retention schedule
-- BiometricMatchSucceeded    — successful biometric verification
-- BiometricMatchFailed       — failed biometric verification
```

## Event Payload Examples

```sql
-- TimeEntryRecorded
-- {
--   "employee_id": "uuid",
--   "entry_type": "clock_in",
--   "punch_time": "2026-05-12T08:02:33-05:00",
--   "source": "mobile_app",
--   "location_id": "uuid",
--   "latitude": 41.8781136,
--   "longitude": -87.6297982,
--   "geofence_status": "inside",
--   "biometric_method": "facial_recognition",
--   "biometric_confidence": 0.987,
--   "device_id": "iPhone14,2-ABCD1234"
-- }

-- TimesheetApproved
-- {
--   "timesheet_id": "uuid",
--   "employee_id": "uuid",
--   "period_start": "2026-05-04",
--   "period_end": "2026-05-10",
--   "total_regular_minutes": 2400,
--   "total_overtime_minutes": 120,
--   "approved_by": "uuid",
--   "approval_notes": "Verified shift differential for night shifts"
-- }

-- ComplianceViolationDetected
-- {
--   "employee_id": "uuid",
--   "rule_type": "max_weekly_hours",
--   "regulation": "EU WTD Art. 6",
--   "threshold_minutes": 2880,
--   "actual_minutes": 2940,
--   "violation_week_start": "2026-05-04",
--   "severity": "violation"
-- }

-- BiometricConsentGranted
-- {
--   "employee_id": "uuid",
--   "consent_type": "facial_recognition",
--   "purpose": "Time and attendance verification via facial recognition",
--   "retention_period_days": 1095,
--   "consent_text": "I consent to the collection and storage of my facial geometry...",
--   "jurisdiction_law": "BIPA",
--   "consent_method": "electronic_signature"
-- }
```

## Read Model Projections (Query Side)

```sql
-- ============================================================
-- PROJECTION: Current Employee State
-- ============================================================
-- Rebuilt from: EmployeeOnboarded, EmployeeUpdated,
-- EmployeeLocationAssigned, EmployeeTerminated events
-- ============================================================

CREATE TABLE proj_employee (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    employee_number VARCHAR(50) NOT NULL,
    email           VARCHAR(255),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    employment_status VARCHAR(20) NOT NULL,
    employment_type VARCHAR(20) NOT NULL,
    flsa_status     VARCHAR(10) NOT NULL,
    primary_location_id UUID,
    primary_department_id UUID,
    jurisdiction_id UUID,
    manager_id      UUID,
    hire_date       DATE NOT NULL,
    termination_date DATE,
    last_event_version INTEGER NOT NULL,                -- track projection currency
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_employee_tenant ON proj_employee (tenant_id);
CREATE INDEX idx_proj_employee_status ON proj_employee (tenant_id, employment_status);

-- ============================================================
-- PROJECTION: Current Timesheet
-- ============================================================
-- Rebuilt from: TimesheetCreated, TimesheetLineUpdated,
-- TimesheetSubmitted, TimesheetApproved, TimesheetRejected events
-- ============================================================

CREATE TABLE proj_timesheet (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    employee_id     UUID NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    period_type     VARCHAR(10) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    total_regular_minutes   INTEGER NOT NULL DEFAULT 0,
    total_overtime_minutes  INTEGER NOT NULL DEFAULT 0,
    total_doubletime_minutes INTEGER NOT NULL DEFAULT 0,
    total_break_minutes     INTEGER NOT NULL DEFAULT 0,
    total_pto_minutes       INTEGER NOT NULL DEFAULT 0,
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    approved_by     UUID,
    rejected_at     TIMESTAMPTZ,
    rejection_reason TEXT,
    last_event_version INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, period_start, period_type)
);

CREATE INDEX idx_proj_timesheet_tenant ON proj_timesheet (tenant_id, status);
CREATE INDEX idx_proj_timesheet_employee ON proj_timesheet (employee_id, period_start);

-- ============================================================
-- PROJECTION: Timesheet Daily Detail
-- ============================================================
CREATE TABLE proj_timesheet_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timesheet_id    UUID NOT NULL REFERENCES proj_timesheet(id),
    work_date       DATE NOT NULL,
    location_id     UUID,
    department_id   UUID,
    project_code    VARCHAR(50),
    labor_category  VARCHAR(50),
    regular_minutes INTEGER NOT NULL DEFAULT 0,
    overtime_minutes INTEGER NOT NULL DEFAULT 0,
    doubletime_minutes INTEGER NOT NULL DEFAULT 0,
    break_minutes   INTEGER NOT NULL DEFAULT 0,
    first_clock_in  TIMESTAMPTZ,
    last_clock_out  TIMESTAMPTZ,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_tsl_timesheet ON proj_timesheet_line (timesheet_id);

-- ============================================================
-- PROJECTION: Live Attendance (who is clocked in right now)
-- ============================================================
-- Rebuilt from: TimeEntryRecorded events
-- Updated in near real-time for dashboard display
-- ============================================================

CREATE TABLE proj_live_attendance (
    employee_id     UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    current_status  VARCHAR(20) NOT NULL,               -- clocked_in, on_break, clocked_out
    clock_in_time   TIMESTAMPTZ,
    last_event_time TIMESTAMPTZ NOT NULL,
    location_id     UUID,
    shift_id        UUID,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_live_tenant ON proj_live_attendance (tenant_id, current_status);

-- ============================================================
-- PROJECTION: Schedule / Shifts
-- ============================================================
-- Rebuilt from: ScheduleCreated, ShiftAssigned, ShiftSwap* events
-- ============================================================

CREATE TABLE proj_shift (
    id              UUID PRIMARY KEY,
    schedule_id     UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    employee_id     UUID,
    location_id     UUID NOT NULL,
    department_id   UUID,
    shift_date      DATE NOT NULL,
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    break_minutes   INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL,
    required_skill_id UUID,
    ical_uid        VARCHAR(255),
    last_event_version INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_shift_employee ON proj_shift (employee_id, shift_date);
CREATE INDEX idx_proj_shift_location ON proj_shift (location_id, shift_date);
CREATE INDEX idx_proj_shift_open ON proj_shift (schedule_id) WHERE employee_id IS NULL;

-- ============================================================
-- PROJECTION: Leave Balances
-- ============================================================
-- Rebuilt from: LeaveAccrued, LeaveApproved, LeaveBalanceAdjusted events
-- ============================================================

CREATE TABLE proj_leave_balance (
    employee_id     UUID NOT NULL,
    leave_type_id   UUID NOT NULL,
    balance_hours   NUMERIC(10,2) NOT NULL DEFAULT 0,
    accrued_ytd     NUMERIC(10,2) NOT NULL DEFAULT 0,
    used_ytd        NUMERIC(10,2) NOT NULL DEFAULT 0,
    year            SMALLINT NOT NULL,
    last_event_version INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (employee_id, leave_type_id, year)
);

-- ============================================================
-- PROJECTION: Compliance Dashboard
-- ============================================================
-- Rebuilt from: ComplianceThresholdApproaching,
-- ComplianceViolationDetected, ComplianceAlertResolved events
-- ============================================================

CREATE TABLE proj_compliance_status (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    employee_id     UUID NOT NULL,
    rule_type       VARCHAR(50) NOT NULL,
    current_value   INTEGER,                            -- e.g., current weekly minutes
    threshold_value INTEGER,                            -- e.g., max weekly minutes
    status          VARCHAR(20) NOT NULL,               -- ok, approaching, violation
    last_alert_id   UUID,
    week_start      DATE NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, rule_type, week_start)
);

CREATE INDEX idx_proj_compliance_tenant ON proj_compliance_status (tenant_id, status);

-- ============================================================
-- PROJECTION: Biometric Consent Registry
-- ============================================================
-- Rebuilt from: BiometricConsentGranted, BiometricConsentRevoked events
-- ============================================================

CREATE TABLE proj_biometric_consent (
    employee_id     UUID NOT NULL,
    consent_type    VARCHAR(30) NOT NULL,
    is_active       BOOLEAN NOT NULL,
    granted_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    jurisdiction_law VARCHAR(50),
    retention_period_days INTEGER,
    last_event_version INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (employee_id, consent_type)
);
```

## Reference Data (Shared Between Write and Read)

```sql
-- ============================================================
-- These tables store reference/configuration data that is NOT
-- event-sourced. They are shared by both write and read sides.
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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

CREATE TABLE department (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    location_id     UUID REFERENCES location(id),
    parent_id       UUID REFERENCES department(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    cost_center     VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    name            VARCHAR(255) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pay_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    jurisdiction_id UUID REFERENCES jurisdiction(id),
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

CREATE TABLE leave_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    category        VARCHAR(30) NOT NULL,
    is_fmla_qualifying BOOLEAN NOT NULL DEFAULT false,
    is_protected    BOOLEAN NOT NULL DEFAULT false,
    accrual_method  VARCHAR(20),
    accrual_rate    NUMERIC(10,4),
    max_balance     NUMERIC(10,2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE skill (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),
    requires_expiry BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE compliance_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    rule_type       VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    threshold_value INTEGER,
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning',
    regulation_reference VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Point-in-Time Reconstruction Query

```sql
-- ============================================================
-- EXAMPLE: Reconstruct timesheet state as it was on March 15, 2026
-- ============================================================
-- This is the key advantage of event sourcing: temporal queries
-- that are impossible with a mutable state model.
-- ============================================================

-- Get all events for a timesheet up to a specific point in time
SELECT
    event_type,
    payload,
    metadata,
    recorded_at
FROM event_store
WHERE stream_id = '{{timesheet_uuid}}'
  AND stream_type = 'Timesheet'
  AND recorded_at <= '2026-03-15T23:59:59Z'
ORDER BY event_version ASC;

-- The application replays these events through the Timesheet aggregate
-- to reconstruct the exact state at that moment:
--   1. TimesheetCreated → initial empty timesheet
--   2. TimeEntryRecorded × N → accumulate hours
--   3. TimesheetSubmitted → status = submitted
--   4. (no TimesheetApproved event before Mar 15) → still submitted

-- ============================================================
-- EXAMPLE: Find all overtime anomalies in the last 90 days
-- for AI-driven pattern detection
-- ============================================================

SELECT
    payload->>'employee_id' AS employee_id,
    payload->>'actual_minutes' AS actual_weekly_minutes,
    payload->>'threshold_minutes' AS threshold,
    event_timestamp,
    metadata->>'source' AS detection_source
FROM event_store
WHERE event_type = 'ComplianceViolationDetected'
  AND tenant_id = '{{tenant_uuid}}'
  AND event_timestamp >= now() - INTERVAL '90 days'
ORDER BY event_timestamp DESC;

-- ============================================================
-- EXAMPLE: Audit trail for a specific time entry
-- ============================================================

SELECT
    event_type,
    payload,
    metadata->>'actor_id' AS changed_by,
    metadata->>'source' AS change_source,
    recorded_at
FROM event_store
WHERE stream_id = '{{time_entry_uuid}}'
  AND stream_type = 'TimeEntry'
ORDER BY event_version ASC;

-- Returns the complete history:
-- v1: TimeEntryRecorded (employee clocked in at 8:02 via mobile)
-- v2: TimeEntryAdjusted (manager changed to 8:00, reason: "rounding per policy")
-- v3: TimeEntryFlagged (system flagged: geofence outside threshold)
-- v4: TimeEntryFlagResolved (manager resolved: "employee was at loading dock")
```

## GDPR Crypto-Shredding Pattern

```sql
-- ============================================================
-- ENCRYPTION KEY TABLE (for crypto-shredding)
-- ============================================================
-- Each employee gets a unique encryption key. To "delete" an
-- employee under GDPR Article 17, destroy their key. All events
-- containing their PII become unreadable, but the event store
-- structure remains intact for aggregate analytics.
-- ============================================================

CREATE TABLE encryption_key (
    employee_id     UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    key_material    BYTEA NOT NULL,                     -- AES-256 key, itself encrypted by master key
    key_version     INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    destroyed_at    TIMESTAMPTZ,                        -- set when GDPR erasure performed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- When GDPR erasure is requested:
-- 1. Set is_active = false, destroyed_at = now()
-- 2. Overwrite key_material with zeros
-- 3. All events for this employee's streams become undecryptable
-- 4. Aggregate/anonymous projections remain valid
-- 5. The event store sequence and structure is preserved
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (Write) | 3 | event_store, event_snapshot, processed_event |
| Reference Data (Shared) | 8 | tenant, location, department, jurisdiction, pay_rule, leave_type, skill, compliance_rule |
| Projections (Read) | 8 | proj_employee, proj_timesheet, proj_timesheet_line, proj_live_attendance, proj_shift, proj_leave_balance, proj_compliance_status, proj_biometric_consent |
| Security | 1 | encryption_key |
| **Total** | **~20** | Fewer tables than normalized model; projections can be added/rebuilt without schema migration |

---

## Key Design Decisions

1. **Single event_store table as source of truth.** All domain state changes are captured as immutable events. This provides a complete, tamper-evident audit trail that satisfies DCAA, FLSA, and GDPR compliance requirements by design.

2. **Stream-per-aggregate pattern.** Each aggregate root (employee, timesheet, schedule, leave request) has its own stream identified by `stream_id` + `stream_type`. This allows efficient replay and supports the natural aggregate boundaries in the T&A domain.

3. **Projections are disposable.** Every `proj_*` table can be dropped and rebuilt from the event store. This means schema changes to read models require no data migration — just rebuild the projection. New reporting requirements can be met by adding new projections without touching the event store.

4. **Optimistic concurrency via event_version.** The unique constraint on `(stream_id, event_version)` prevents write conflicts. Two concurrent writes to the same aggregate will fail — the second writer must re-read and retry. This eliminates lost updates without database-level locking.

5. **Crypto-shredding for GDPR erasure.** Rather than deleting events (which would break the event store's integrity), employee-specific encryption keys are destroyed, rendering PII in events unreadable while preserving the event sequence for aggregate analytics and compliance.

6. **Reference data is NOT event-sourced.** Configuration data (tenants, locations, pay rules, leave types) changes infrequently and doesn't need an audit trail. Keeping it in regular mutable tables avoids unnecessary complexity.

7. **Event payload versioning via schema_version.** When event payloads evolve (adding fields, renaming fields), the `schema_version` field enables upcasting: the projection layer transforms old event formats to new formats during replay without modifying stored events.

8. **Partitioned event store for scale.** The event_store table should be partitioned by `recorded_at` (monthly) in production. This enables efficient time-range queries, archival of old partitions, and prevents a single massive table from degrading performance.

9. **Near-real-time live attendance projection.** The `proj_live_attendance` table is updated with minimal latency from `TimeEntryRecorded` events, providing the "who is clocked in right now" dashboard view that managers expect.

10. **AI analytics on raw event streams.** The event store is a natural data source for machine learning models that detect buddy-punching patterns, predict absence trends, or optimize scheduling. Raw events contain richer signal than aggregated projections.
