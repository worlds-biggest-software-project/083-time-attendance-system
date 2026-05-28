# Time & Attendance System -- Development Plan

> Project: 083-time-attendance-system
> Generated: 2026-05-25
> Status: Planning

---

## Table of Contents

1. [Technology Decisions](#technology-decisions)
2. [Project Structure](#project-structure)
3. [Phase Dependency Graph](#phase-dependency-graph)
4. [Phase 1: Foundation & Core Data Model](#phase-1-foundation--core-data-model)
5. [Phase 2: Time Capture & Clock-In](#phase-2-time-capture--clock-in)
6. [Phase 3: Overtime Calculation & Pay Rules Engine](#phase-3-overtime-calculation--pay-rules-engine)
7. [Phase 4: Timesheet Approval Workflow](#phase-4-timesheet-approval-workflow)
8. [Phase 5: Employee Self-Service & Web UI](#phase-5-employee-self-service--web-ui)
9. [Phase 6: Leave & Absence Management](#phase-6-leave--absence-management)
10. [Phase 7: Scheduling Engine](#phase-7-scheduling-engine)
11. [Phase 8: Compliance Engine & Alerting](#phase-8-compliance-engine--alerting)
12. [Phase 9: Payroll Export & Integrations](#phase-9-payroll-export--integrations)
13. [Phase 10: Biometric Clock-In & Data Sovereignty](#phase-10-biometric-clock-in--data-sovereignty)
14. [Phase 11: AI-Powered Scheduling & NLP](#phase-11-ai-powered-scheduling--nlp)
15. [Phase 12: Mobile App & Kiosk Mode](#phase-12-mobile-app--kiosk-mode)
16. [Definition of Done](#definition-of-done)

---

## Technology Decisions

### Database: PostgreSQL 16+ with Hybrid Relational + JSONB (Data Model Suggestion 3)

**Rationale:** After evaluating all four proposed data models, the Hybrid Relational + JSONB model (Suggestion 3) is the best fit for this project for the following reasons:

1. **Multi-jurisdiction flexibility without schema migrations.** The system must support FLSA (US federal), 50 US state variants (California daily OT, etc.), EU Working Time Directive (27 member states), and Australian modern awards. The JSONB approach handles jurisdiction-specific pay rules, break requirements, and compliance fields as configuration rather than schema changes.

2. **Lower table count accelerates MVP delivery.** At ~23 tables versus ~36 (normalized) or ~21+graph, the hybrid model reduces migration complexity while maintaining relational integrity for hot-path queries.

3. **GIN-indexed JSONB queries perform well for the expected data volume.** SMB target (5-500 employees) means query volumes are manageable. GIN indexes on compliance_data, flags, and profile columns provide sub-millisecond containment queries.

4. **Event sourcing (Suggestion 2) deferred to Phase 8.** The compliance audit trail is critical but can be implemented as an append-only `audit_event` table layered on top of the hybrid model, rather than redesigning the entire data layer around CQRS. This gets audit capabilities without the engineering overhead of full event sourcing.

5. **Graph relationships (Suggestion 4) deferred.** Manager hierarchies and approval chains are initially implemented via `manager_id` FK and recursive CTEs. If relationship complexity grows (credential-matched scheduling across 1000+ employees), the graph_node/graph_edge tables can be added later.

**Rejected alternatives:**
- Suggestion 1 (fully normalized): 45-55 tables is excessive for MVP; adding jurisdiction fields requires schema changes
- Suggestion 2 (event-sourced CQRS): Engineering overhead too high for initial team; eventual consistency complicates MVP UX
- Suggestion 4 (graph-relational): Graph layer adds infrastructure complexity; benefits emerge at scale the MVP won't reach

### Backend: Node.js 22 LTS + TypeScript 5.x + Fastify

**Rationale:**
- Fastify chosen over Express for built-in JSON Schema validation (aligns with JSONB schema validation requirement), 2-3x higher throughput, and native TypeScript support
- Node.js selected for the large ecosystem of payroll/HRIS integration libraries, async I/O for webhook handling, and developer hiring pool
- TypeScript for type safety across the JSONB schema boundaries -- JSON Schema types can be generated from TypeScript interfaces

### ORM/Query Layer: Drizzle ORM

**Rationale:**
- SQL-first approach aligns with the complex overtime calculation queries and JSONB operations needed
- Generates type-safe PostgreSQL queries including JSONB operators (`->`, `->>`, `@>`)
- Zero-overhead migrations via SQL files (not auto-generated), giving full control over jurisdiction-specific data seeding
- Lighter weight than Prisma; better PostgreSQL-specific feature support (RLS, GIN indexes, JSONB)

### Frontend: Next.js 15 (App Router) + React 19 + Tailwind CSS + shadcn/ui

**Rationale:**
- Server Components reduce client bundle size for the data-heavy timesheet and schedule views
- App Router enables nested layouts critical for the manager dashboard (left nav + content + detail panel)
- shadcn/ui provides accessible, customizable components without vendor lock-in (important for the OSS positioning)
- Tailwind for rapid UI iteration; TimeTrex's dated UI is a known competitive weakness to exploit

### Authentication: Better Auth (self-hosted) + OpenID Connect

**Rationale:**
- Self-hosted authentication aligns with the data sovereignty positioning
- Better Auth provides OAuth 2.0 / OIDC provider capabilities out of the box, enabling SSO with Azure AD/Entra ID, Okta, and Google Workspace
- WebAuthn/FIDO2 support for passwordless kiosk authentication (eliminates PIN-sharing buddy-punch vector)

### AI/ML: Claude API (Anthropic) for NLP scheduling; local inference for anomaly detection

**Rationale:**
- Claude API for natural language schedule requests ("Schedule 3 welders for night shift Tuesday, avoid overtime") -- requires reasoning over constraints
- Local anomaly detection (TensorFlow.js or ONNX Runtime) for buddy-punching pattern detection and attendance prediction -- must run on-premises for data sovereignty
- MCP server exposure for AI assistant integration (first-mover opportunity identified in standards research)

### Deployment: Docker Compose (self-hosted primary) + Kubernetes Helm chart (cloud option)

**Rationale:**
- Docker Compose is the simplest self-hosted deployment for the SMB target market
- Biometric data processing must run on-premises; containerized deployment makes this practical
- Kubernetes Helm chart for organizations with existing K8s infrastructure
- Cloud-hosted option (managed) deferred to post-MVP

### API Design: REST + OpenAPI 3.1 (design-first)

**Rationale:**
- OpenAPI spec written before implementation (standards research recommends design-first approach)
- REST chosen over GraphQL for simpler caching, broader integration ecosystem, and payroll system compatibility
- HR Open Standards HR-JSON Timecard schema adopted for payroll export endpoints
- MCP server endpoints added alongside REST for AI assistant integration

---

## Project Structure

```
time-attendance-system/
├── apps/
│   ├── api/                          # Fastify backend
│   │   ├── src/
│   │   │   ├── modules/
│   │   │   │   ├── auth/             # Authentication & RBAC
│   │   │   │   ├── tenant/           # Multi-tenancy
│   │   │   │   ├── employee/         # Employee CRUD
│   │   │   │   ├── time-entry/       # Clock-in/out, punches
│   │   │   │   ├── timesheet/        # Aggregation & approval
│   │   │   │   ├── pay-rule/         # Overtime & pay configuration
│   │   │   │   ├── scheduling/       # Shifts, schedules, swaps
│   │   │   │   ├── leave/            # Leave requests, balances, FMLA
│   │   │   │   ├── compliance/       # Rules engine, alerts
│   │   │   │   ├── biometric/        # Templates, consent, matching
│   │   │   │   ├── payroll-export/   # Export generation
│   │   │   │   ├── notification/     # In-app, email, push
│   │   │   │   └── ai/              # NLP scheduling, anomaly detection
│   │   │   ├── shared/
│   │   │   │   ├── db/              # Drizzle schema, migrations
│   │   │   │   ├── middleware/      # Auth, tenant isolation, rate limiting
│   │   │   │   ├── validators/      # JSON Schema validators for JSONB
│   │   │   │   └── utils/           # Date/time, timezone, currency
│   │   │   └── server.ts
│   │   ├── drizzle/                  # Migration files
│   │   ├── openapi/                  # OpenAPI 3.1 spec (design-first)
│   │   └── package.json
│   ├── web/                          # Next.js frontend
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── (auth)/          # Login, SSO
│   │   │   │   ├── (dashboard)/     # Manager dashboard
│   │   │   │   ├── (employee)/      # Employee self-service
│   │   │   │   ├── (admin)/         # Tenant admin
│   │   │   │   └── api/             # Next.js API routes (BFF)
│   │   │   ├── components/
│   │   │   │   ├── ui/              # shadcn/ui primitives
│   │   │   │   ├── timesheet/
│   │   │   │   ├── schedule/
│   │   │   │   ├── clock/
│   │   │   │   └── compliance/
│   │   │   └── lib/
│   │   └── package.json
│   └── mobile/                       # React Native (Phase 12)
├── packages/
│   ├── overtime-engine/              # Standalone FLSA/WTD calculation library
│   ├── compliance-rules/             # Rule definitions per jurisdiction
│   ├── biometric-client/             # On-device facial recognition SDK
│   ├── payroll-formats/              # CSV, HR-JSON, QuickBooks, Xero formatters
│   ├── ical-export/                  # RFC 5545 schedule export
│   └── shared-types/                 # TypeScript types shared across apps
├── docker/
│   ├── docker-compose.yml
│   ├── docker-compose.dev.yml
│   └── Dockerfile.*
├── helm/                             # Kubernetes Helm chart
├── docs/
│   ├── openapi.yaml                  # Master OpenAPI spec
│   ├── json-schemas/                 # JSONB column validation schemas
│   └── compliance/                   # Jurisdiction rule documentation
├── turbo.json                        # Turborepo config
├── package.json
└── README.md
```

---

## Phase Dependency Graph

```
Phase 1: Foundation & Core Data Model
  │
  ├──> Phase 2: Time Capture & Clock-In
  │      │
  │      ├──> Phase 3: Overtime Calculation & Pay Rules Engine
  │      │      │
  │      │      └──> Phase 4: Timesheet Approval Workflow
  │      │             │
  │      │             ├──> Phase 9: Payroll Export & Integrations
  │      │             │
  │      │             └──> Phase 5: Employee Self-Service & Web UI
  │      │
  │      └──> Phase 8: Compliance Engine & Alerting (depends on Phase 3 too)
  │
  ├──> Phase 6: Leave & Absence Management (depends on Phase 1 only)
  │      │
  │      └──> Phase 7: Scheduling Engine (depends on Phase 6 + Phase 2)
  │             │
  │             └──> Phase 11: AI-Powered Scheduling & NLP
  │
  ├──> Phase 10: Biometric Clock-In & Data Sovereignty (depends on Phase 2)
  │
  └──> Phase 12: Mobile App & Kiosk Mode (depends on Phase 2 + Phase 5)
```

**Critical path:** Phase 1 -> Phase 2 -> Phase 3 -> Phase 4 -> Phase 5 (MVP release)

**Parallel tracks after Phase 1:**
- Track A (Critical): Phases 2 -> 3 -> 4 -> 5 (time capture through employee UI)
- Track B (Leave): Phase 6 (can start after Phase 1, joins scheduling in Phase 7)
- Track C (Compliance): Phase 8 (can start after Phase 3)

---

## Phase 1: Foundation & Core Data Model

**Goal:** Establish the monorepo structure, database schema, authentication system, multi-tenancy, and API skeleton so that all subsequent phases build on a solid foundation.

**Duration estimate:** 3-4 weeks

### Task 1.1: Monorepo & Tooling Setup

**What:** Initialize the Turborepo monorepo with the api (Fastify), web (Next.js), and shared packages. Configure TypeScript, ESLint, Prettier, and CI pipeline.

**Design:**

```typescript
// turbo.json
{
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["^build"] },
    "lint": {},
    "typecheck": { "dependsOn": ["^build"] },
    "db:migrate": { "cache": false }
  }
}

// apps/api/package.json (key dependencies)
{
  "dependencies": {
    "fastify": "^5.x",
    "@fastify/cors": "^10.x",
    "@fastify/helmet": "^13.x",
    "@fastify/rate-limit": "^10.x",
    "@fastify/swagger": "^9.x",
    "drizzle-orm": "^0.36.x",
    "postgres": "^3.4.x",
    "zod": "^3.23.x",
    "better-auth": "^1.x"
  }
}
```

**Testing:**
- `turbo build` completes without errors across all packages
- `turbo lint` and `turbo typecheck` pass
- `turbo dev` starts api on port 3001 and web on port 3000 simultaneously
- CI pipeline (GitHub Actions) runs build, lint, typecheck, and test on every PR
- Package dependency graph is correct (shared-types builds before api and web)

### Task 1.2: Database Schema & Migrations

**What:** Implement the Hybrid Relational + JSONB schema (Data Model Suggestion 3) in Drizzle ORM. Create initial migration files. Seed reference data for US jurisdictions and FLSA pay rules.

**Design:**

```typescript
// packages/shared-types/src/db/schema/tenant.ts
import { pgTable, uuid, varchar, jsonb, timestamp, boolean } from 'drizzle-orm/pg-core';

export const tenant = pgTable('tenant', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  timezone: varchar('timezone', { length: 50 }).notNull().default('UTC'),
  locale: varchar('locale', { length: 10 }).notNull().default('en-US'),
  subscriptionPlan: varchar('subscription_plan', { length: 50 }).notNull().default('free'),
  config: jsonb('config').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// apps/api/src/shared/db/schema/employee.ts
export const employee = pgTable('employee', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenant.id),
  employeeNumber: varchar('employee_number', { length: 50 }).notNull(),
  email: varchar('email', { length: 255 }),
  firstName: varchar('first_name', { length: 100 }).notNull(),
  lastName: varchar('last_name', { length: 100 }).notNull(),
  hireDate: date('hire_date').notNull(),
  terminationDate: date('termination_date'),
  employmentStatus: varchar('employment_status', { length: 20 }).notNull().default('active'),
  employmentType: varchar('employment_type', { length: 20 }).notNull().default('full_time'),
  flsaStatus: varchar('flsa_status', { length: 10 }).notNull().default('non_exempt'),
  primaryLocationId: uuid('primary_location_id').references(() => location.id),
  primaryDepartmentId: uuid('primary_department_id').references(() => department.id),
  managerId: uuid('manager_id').references((): AnyPgColumn => employee.id),
  hourlyRate: numeric('hourly_rate', { precision: 10, scale: 4 }),
  currencyCode: char('currency_code', { length: 3 }).default('USD'),
  profile: jsonb('profile').notNull().default({}),
  payConfig: jsonb('pay_config').notNull().default({}),
  complianceData: jsonb('compliance_data').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueEmployeeNumber: unique().on(table.tenantId, table.employeeNumber),
  tenantIdx: index('idx_employee_tenant').on(table.tenantId),
  statusIdx: index('idx_employee_status').on(table.tenantId, table.employmentStatus),
  profileGin: index('idx_employee_profile').using('gin', table.profile),
  complianceGin: index('idx_employee_compliance').using('gin', table.complianceData),
}));

// Seed data: FLSA default pay rule
// drizzle/seeds/001-flsa-pay-rules.sql
INSERT INTO pay_rule (tenant_id, name, country_code, workweek_start_day,
  overtime_weekly_threshold, overtime_multiplier, rules, is_default, is_active)
VALUES
  (NULL, 'FLSA Federal Default', 'US', 0, 2400, 1.50, '{}', true, true),
  (NULL, 'California', 'US', 0, 2400, 1.50,
   '{"overtime_daily_threshold":480,"doubletime_daily_threshold":720,
     "doubletime_multiplier":2.00,"seventh_day_overtime":true}', false, true);
```

**Testing:**
- Migration applies cleanly to a fresh PostgreSQL 16 database: `drizzle-kit push` succeeds
- Migration rollback works: `drizzle-kit drop` removes all tables
- All foreign key constraints are enforced: inserting an employee with a non-existent tenant_id fails
- GIN indexes exist on profile, compliance_data, and flags columns (verify with `\di` in psql)
- Seed data for FLSA federal and California pay rules is present after migration
- JSONB columns accept arbitrary JSON without constraint violations
- Unique constraint on (tenant_id, employee_number) prevents duplicate employee numbers within a tenant

### Task 1.3: Multi-Tenancy & Row-Level Security

**What:** Implement tenant isolation using Drizzle middleware that automatically injects `tenant_id` filters on all queries. Optionally enable PostgreSQL RLS policies as a defense-in-depth measure.

**Design:**

```typescript
// apps/api/src/shared/middleware/tenant-context.ts
import { FastifyRequest } from 'fastify';

export interface TenantContext {
  tenantId: string;
  employeeId: string;
  roles: string[];
}

// Fastify preHandler hook that extracts tenant context from JWT
export async function tenantContextHook(request: FastifyRequest) {
  const claims = request.user; // populated by auth middleware
  request.tenantContext = {
    tenantId: claims.tenant_id,
    employeeId: claims.employee_id,
    roles: claims.roles,
  };
}

// Drizzle query wrapper that enforces tenant isolation
export function withTenant<T>(
  db: DrizzleDB,
  tenantId: string,
  queryFn: (db: DrizzleDB) => Promise<T>
): Promise<T> {
  // All queries within queryFn automatically include WHERE tenant_id = tenantId
  return db.transaction(async (tx) => {
    await tx.execute(sql`SET LOCAL app.current_tenant_id = ${tenantId}`);
    return queryFn(tx);
  });
}

// PostgreSQL RLS policy (defense-in-depth)
// drizzle/migrations/002-rls-policies.sql
ALTER TABLE employee ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON employee
  USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

**Testing:**
- API requests without a valid tenant context return 401 Unauthorized
- An employee from Tenant A cannot read or modify data belonging to Tenant B
- Direct SQL queries with `SET app.current_tenant_id` to Tenant A only return Tenant A rows
- RLS policies are enforced even if application-level filtering has a bug (insert a row as Tenant A, verify Tenant B session cannot SELECT it via RLS)
- Superuser/admin can bypass RLS for cross-tenant operations (support/debugging)

### Task 1.4: Authentication & RBAC

**What:** Configure Better Auth for local email/password authentication and OpenID Connect SSO. Implement scoped RBAC (global, location, department scopes) with the three built-in roles: admin, manager, employee.

**Design:**

```typescript
// apps/api/src/modules/auth/auth.config.ts
import { betterAuth } from 'better-auth';
import { drizzleAdapter } from 'better-auth/adapters/drizzle';

export const auth = betterAuth({
  database: drizzleAdapter(db, { provider: 'pg' }),
  emailAndPassword: { enabled: true },
  socialProviders: {
    // OIDC for enterprise SSO
    oidc: {
      clientId: process.env.OIDC_CLIENT_ID,
      clientSecret: process.env.OIDC_CLIENT_SECRET,
      issuer: process.env.OIDC_ISSUER,
    },
  },
  session: {
    strategy: 'jwt',
    maxAge: 8 * 60 * 60, // 8 hours (typical shift length)
  },
});

// RBAC permission check
// apps/api/src/shared/middleware/rbac.ts
export function requirePermission(resource: string, action: string) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const { roles, tenantId } = request.tenantContext;
    const employeeRoles = await db.select()
      .from(employeeRole)
      .innerJoin(appRole, eq(employeeRole.roleId, appRole.id))
      .where(eq(employeeRole.employeeId, request.tenantContext.employeeId));

    const hasPermission = employeeRoles.some(r => {
      const perms = r.app_role.permissions as Permission[];
      return perms.some(p =>
        p.resource === resource &&
        p.actions.includes(action) &&
        matchesScope(p.scope, request)
      );
    });

    if (!hasPermission) {
      reply.code(403).send({ error: 'Insufficient permissions' });
    }
  };
}
```

**Testing:**
- Email/password registration creates an employee record and returns a JWT
- JWT contains tenant_id, employee_id, and role claims
- OIDC flow redirects to the configured identity provider and returns with a valid session
- Admin role can access all endpoints for their tenant
- Manager role can approve timesheets only for employees in their location/department scope
- Employee role can only read/modify their own time entries and profile
- Expired JWT (>8 hours) returns 401
- Invalid JWT signature returns 401
- RBAC permission check returns 403 when an employee tries to access a manager-only endpoint

### Task 1.5: API Skeleton & OpenAPI Spec

**What:** Create the design-first OpenAPI 3.1 specification for all MVP endpoints. Generate route stubs from the spec. Configure Fastify Swagger for auto-generated API documentation.

**Design:**

```yaml
# docs/openapi.yaml (excerpt)
openapi: 3.1.0
info:
  title: Time & Attendance System API
  version: 0.1.0
  description: Open-source time capture, scheduling, and compliance platform
paths:
  /api/v1/time-entries:
    post:
      operationId: createTimeEntry
      summary: Record a clock-in, clock-out, or break event
      tags: [Time Capture]
      security: [{ bearerAuth: [] }]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateTimeEntryRequest'
      responses:
        '201':
          description: Time entry recorded
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/TimeEntry'
        '400':
          $ref: '#/components/responses/ValidationError'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '409':
          description: Duplicate entry (already clocked in)

components:
  schemas:
    CreateTimeEntryRequest:
      type: object
      required: [entry_type, punch_time, source]
      properties:
        entry_type:
          type: string
          enum: [clock_in, clock_out, break_start, break_end]
        punch_time:
          type: string
          format: date-time
        source:
          type: string
          enum: [web, mobile_app, kiosk, manual, api]
        location_id:
          type: string
          format: uuid
        geolocation:
          $ref: '#/components/schemas/Geolocation'
        biometric:
          $ref: '#/components/schemas/BiometricVerification'
        work_allocation:
          $ref: '#/components/schemas/WorkAllocation'
```

**Testing:**
- OpenAPI spec validates without errors using `swagger-cli validate docs/openapi.yaml`
- Fastify Swagger UI is accessible at `/documentation` in development
- All route stubs return 501 Not Implemented (confirming routes are registered)
- Request validation rejects payloads that violate the OpenAPI schema (missing required fields, wrong types)
- Response serialization strips fields not defined in the response schema (preventing data leakage)

### Task 1.6: Employee CRUD

**What:** Implement full CRUD operations for the employee entity, including JSONB profile, payConfig, and complianceData management. This is the first module that exercises the full stack (API route -> service -> Drizzle query -> PostgreSQL).

**Design:**

```typescript
// apps/api/src/modules/employee/employee.service.ts
export class EmployeeService {
  async create(tenantId: string, data: CreateEmployeeInput): Promise<Employee> {
    // Validate JSONB fields against JSON Schema
    validateJsonSchema(data.profile, profileSchema);
    validateJsonSchema(data.payConfig, payConfigSchema);
    validateJsonSchema(data.complianceData, complianceDataSchema);

    const [emp] = await db.insert(employee).values({
      tenantId,
      employeeNumber: data.employeeNumber,
      email: data.email,
      firstName: data.firstName,
      lastName: data.lastName,
      hireDate: data.hireDate,
      employmentType: data.employmentType,
      flsaStatus: data.flsaStatus,
      primaryLocationId: data.primaryLocationId,
      primaryDepartmentId: data.primaryDepartmentId,
      managerId: data.managerId,
      hourlyRate: data.hourlyRate,
      profile: data.profile ?? {},
      payConfig: data.payConfig ?? {},
      complianceData: data.complianceData ?? {},
    }).returning();

    return emp;
  }

  async findByTenant(tenantId: string, filters: EmployeeFilters): Promise<Employee[]> {
    let query = db.select().from(employee)
      .where(eq(employee.tenantId, tenantId));

    if (filters.status) {
      query = query.where(eq(employee.employmentStatus, filters.status));
    }
    if (filters.locationId) {
      query = query.where(eq(employee.primaryLocationId, filters.locationId));
    }
    if (filters.jurisdiction) {
      // JSONB query: filter by jurisdiction in compliance_data
      query = query.where(
        sql`${employee.complianceData}->>'jurisdiction' = ${filters.jurisdiction}`
      );
    }

    return query.orderBy(employee.lastName, employee.firstName);
  }
}
```

**Testing:**
- POST /api/v1/employees creates an employee and returns the full record with generated UUID
- GET /api/v1/employees returns only employees for the authenticated tenant
- GET /api/v1/employees?status=active filters correctly
- GET /api/v1/employees?jurisdiction=US-CA filters by JSONB compliance_data field
- PUT /api/v1/employees/:id updates relational and JSONB fields atomically
- DELETE /api/v1/employees/:id soft-deletes (sets employment_status to 'terminated')
- Invalid JSONB profile data (e.g., certifications without required fields) returns 400 validation error
- Creating an employee with a duplicate (tenant_id, employee_number) returns 409 Conflict
- A manager-scoped user can only see employees in their department
- All timestamp fields use ISO 8601 with timezone offset in responses

---

## Phase 2: Time Capture & Clock-In

**Goal:** Implement the core clock-in/clock-out functionality via the web interface, including GPS geofencing validation and the time entry data pipeline.

**Duration estimate:** 3-4 weeks

**Dependencies:** Phase 1 (database schema, auth, employee CRUD)

### Task 2.1: Time Entry API

**What:** Implement POST/GET/PATCH endpoints for time entries. Enforce business rules: cannot clock in if already clocked in, cannot clock out without a preceding clock in, geofence validation against the location's configured radius.

**Design:**

```typescript
// apps/api/src/modules/time-entry/time-entry.service.ts
export class TimeEntryService {
  async recordEntry(tenantId: string, employeeId: string, data: CreateTimeEntryInput): Promise<TimeEntry> {
    // 1. Check current clock status
    const lastEntry = await this.getLastEntry(employeeId);
    if (data.entryType === 'clock_in' && lastEntry?.entryType === 'clock_in') {
      throw new ConflictError('Already clocked in. Clock out first.');
    }
    if (data.entryType === 'clock_out' && lastEntry?.entryType !== 'clock_in') {
      throw new ConflictError('Not currently clocked in.');
    }

    // 2. Validate geofence if location provided
    let geofenceStatus = 'unknown';
    if (data.locationId && data.geolocation) {
      const loc = await this.locationService.findById(data.locationId);
      geofenceStatus = this.validateGeofence(
        data.geolocation.latitude,
        data.geolocation.longitude,
        loc
      );
    }

    // 3. Build flags array
    const flags: TimeEntryFlag[] = [];
    if (geofenceStatus === 'outside') {
      flags.push({
        type: 'outside_geofence',
        detectedAt: new Date().toISOString(),
        resolved: false,
      });
    }

    // 4. Insert time entry
    const [entry] = await db.insert(timeEntry).values({
      tenantId,
      employeeId,
      entryType: data.entryType,
      punchTime: data.punchTime,
      source: data.source,
      locationId: data.locationId,
      geolocation: {
        ...data.geolocation,
        geofenceStatus,
      },
      biometric: data.biometric ?? null,
      workAllocation: data.workAllocation ?? null,
      flags,
      deviceInfo: data.deviceInfo ?? null,
      recordedAt: new Date(),
    }).returning();

    // 5. Emit event for real-time dashboard
    this.eventBus.emit('time_entry.recorded', entry);

    return entry;
  }

  private validateGeofence(
    lat: number, lng: number, location: Location
  ): 'inside' | 'outside' | 'unknown' {
    if (!location.geofence || Object.keys(location.geofence).length === 0) {
      return 'unknown';
    }
    const fence = location.geofence;
    if (fence.type === 'circle') {
      const distance = haversineDistance(
        lat, lng, fence.center.lat, fence.center.lng
      );
      return distance <= fence.radiusMeters ? 'inside' : 'outside';
    }
    // Polygon geofence: point-in-polygon algorithm
    if (fence.type === 'Polygon') {
      return pointInPolygon([lng, lat], fence.coordinates) ? 'inside' : 'outside';
    }
    return 'unknown';
  }
}
```

**Testing:**
- POST clock_in when not clocked in succeeds with 201
- POST clock_in when already clocked in returns 409 Conflict
- POST clock_out when not clocked in returns 409 Conflict
- POST clock_out after clock_in succeeds and records the pair
- Geofence inside location radius: geolocation.geofence_status = 'inside', no flags
- Geofence 200m outside a 100m radius: geolocation.geofence_status = 'outside', flags contains 'outside_geofence'
- Geofence with polygon location: point-in-polygon correctly identifies inside/outside
- No location provided: geofence_status = 'unknown', no flags
- Time entries are ordered by punch_time DESC when retrieved
- Entries from Tenant A are not visible to Tenant B
- DCAA work_allocation JSONB accepts project_code, labor_category, and task_code fields

### Task 2.2: Web Clock-In Interface

**What:** Build the browser-based clock-in/out page. Display current status (clocked in/out, duration), location name, and a prominent clock button. Request browser geolocation on clock-in.

**Design:**

```tsx
// apps/web/src/app/(employee)/clock/page.tsx
'use client';

import { useCallback, useEffect, useState } from 'react';
import { Button } from '@/components/ui/button';
import { Card } from '@/components/ui/card';
import { useCurrentStatus } from '@/hooks/use-current-status';
import { useGeolocation } from '@/hooks/use-geolocation';

export default function ClockPage() {
  const { status, lastEntry, elapsed, refresh } = useCurrentStatus();
  const { position, requestPosition } = useGeolocation();
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleClock = useCallback(async () => {
    setIsSubmitting(true);
    try {
      const geo = await requestPosition();
      const entryType = status === 'clocked_in' ? 'clock_out' : 'clock_in';
      await fetch('/api/v1/time-entries', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          entry_type: entryType,
          punch_time: new Date().toISOString(),
          source: 'web',
          geolocation: geo ? {
            latitude: geo.coords.latitude,
            longitude: geo.coords.longitude,
            accuracy_meters: geo.coords.accuracy,
          } : undefined,
        }),
      });
      refresh();
    } finally {
      setIsSubmitting(false);
    }
  }, [status, requestPosition, refresh]);

  return (
    <Card className="max-w-md mx-auto mt-8 p-6">
      <div className="text-center">
        <p className="text-sm text-muted-foreground">Current Status</p>
        <p className="text-2xl font-bold mt-1">
          {status === 'clocked_in' ? 'Clocked In' : 'Clocked Out'}
        </p>
        {status === 'clocked_in' && (
          <p className="text-4xl font-mono mt-4">{formatElapsed(elapsed)}</p>
        )}
        <Button
          size="lg"
          className="mt-6 w-full h-16 text-xl"
          variant={status === 'clocked_in' ? 'destructive' : 'default'}
          onClick={handleClock}
          disabled={isSubmitting}
        >
          {status === 'clocked_in' ? 'Clock Out' : 'Clock In'}
        </Button>
      </div>
    </Card>
  );
}
```

**Testing:**
- Page renders current clock status correctly for a clocked-out employee
- Clicking "Clock In" requests browser geolocation permission
- After granting geolocation, clock-in succeeds and UI transitions to "Clocked In" with running timer
- Clicking "Clock Out" records the clock-out and UI transitions back to "Clocked Out"
- Denying geolocation still allows clock-in (geolocation is optional) but logs a warning
- Button is disabled during API call to prevent double-submission
- Elapsed timer updates every second when clocked in
- Page works correctly after browser refresh (fetches current status from API)

### Task 2.3: Real-Time Attendance Dashboard

**What:** Build a manager-facing dashboard showing all employees currently clocked in/out, with location and duration. Uses Server-Sent Events (SSE) for live updates without polling.

**Design:**

```typescript
// apps/api/src/modules/time-entry/attendance-stream.ts
export function attendanceStreamRoute(fastify: FastifyInstance) {
  fastify.get('/api/v1/attendance/stream', {
    preHandler: [requirePermission('attendance', 'read')],
  }, async (request, reply) => {
    const { tenantId } = request.tenantContext;

    reply.raw.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      Connection: 'keep-alive',
    });

    // Send initial state
    const currentAttendance = await attendanceService.getCurrentAttendance(tenantId);
    reply.raw.write(`data: ${JSON.stringify({ type: 'snapshot', data: currentAttendance })}\n\n`);

    // Subscribe to live updates
    const handler = (entry: TimeEntry) => {
      if (entry.tenantId === tenantId) {
        reply.raw.write(`data: ${JSON.stringify({ type: 'update', data: entry })}\n\n`);
      }
    };

    eventBus.on('time_entry.recorded', handler);
    request.raw.on('close', () => eventBus.off('time_entry.recorded', handler));
  });
}
```

**Testing:**
- Manager opening dashboard receives initial snapshot of all currently clocked-in employees
- When Employee A clocks in, manager's dashboard updates within 1 second without page refresh
- When Employee A clocks out, their status updates to "Clocked Out" on the dashboard
- Dashboard shows employee name, location, clock-in time, and elapsed duration
- Geofence violation flag is visually indicated (warning icon) on the dashboard
- SSE connection reconnects automatically after network interruption
- Non-manager employees cannot access the attendance stream (403)
- Dashboard correctly handles multiple locations (filter by location)

### Task 2.4: Audit Trail for Time Entries

**What:** Implement the time_entry_audit table and automatic logging of all changes to time entries. Every field change records the old value, new value, who changed it, and why.

**Design:**

```typescript
// apps/api/src/modules/time-entry/time-entry.service.ts
async adjustEntry(
  entryId: string,
  adjustedBy: string,
  changes: Partial<TimeEntryUpdate>,
  reason: string
): Promise<TimeEntry> {
  return db.transaction(async (tx) => {
    const [current] = await tx.select()
      .from(timeEntry)
      .where(eq(timeEntry.id, entryId));

    if (!current) throw new NotFoundError('Time entry not found');

    // Record audit trail for each changed field
    const auditRecords = [];
    for (const [field, newValue] of Object.entries(changes)) {
      if (current[field] !== newValue) {
        auditRecords.push({
          timeEntryId: entryId,
          changes: { [field]: { old: current[field], new: newValue } },
          changedBy: adjustedBy,
          changeReason: reason,
        });
      }
    }

    if (auditRecords.length > 0) {
      await tx.insert(timeEntryAudit).values(auditRecords);
    }

    // Apply changes
    const [updated] = await tx.update(timeEntry)
      .set({ ...changes, updatedAt: new Date() })
      .where(eq(timeEntry.id, entryId))
      .returning();

    return updated;
  });
}
```

**Testing:**
- Adjusting a time entry's punch_time creates an audit record with old and new values
- Audit record includes the ID of the employee who made the change
- Audit record includes the mandatory change reason text
- GET /api/v1/time-entries/:id/audit returns the full change history ordered chronologically
- Audit records cannot be modified or deleted via the API (immutable)
- Adjusting multiple fields in one request creates a single audit record with all changes
- Original time entry retains the adjusted_time field (separate from punch_time)

---

## Phase 3: Overtime Calculation & Pay Rules Engine

**Goal:** Build the standalone overtime calculation engine that computes regular, overtime, and double-time hours based on configurable pay rules per jurisdiction. This is the core compliance differentiator.

**Duration estimate:** 3-4 weeks

**Dependencies:** Phase 2 (time entries exist to calculate against)

### Task 3.1: Overtime Engine Package

**What:** Create the `packages/overtime-engine` standalone library that calculates overtime given a set of time entries and a pay rule configuration. Must handle FLSA federal (40hr/week), California (8hr/day + 12hr double-time + 7th consecutive day), and EU WTD (48hr/week maximum, 11hr daily rest).

**Design:**

```typescript
// packages/overtime-engine/src/index.ts
export interface PayRuleConfig {
  workweekStartDay: 0 | 1 | 2 | 3 | 4 | 5 | 6;  // 0=Sunday
  overtimeWeeklyThresholdMinutes: number;           // FLSA: 2400 (40hrs)
  overtimeMultiplier: number;                       // FLSA: 1.50
  overtimeDailyThresholdMinutes?: number;            // CA: 480 (8hrs)
  doubletimeDailyThresholdMinutes?: number;          // CA: 720 (12hrs)
  doubletimeMultiplier?: number;                     // CA: 2.00
  seventhDayOvertime?: boolean;                      // CA: true
  maxWeeklyMinutes?: number;                         // EU WTD: 2880 (48hrs)
  minDailyRestMinutes?: number;                      // EU WTD: 660 (11hrs)
  minWeeklyRestMinutes?: number;                     // EU WTD: 1440 (24hrs)
  breakRules?: BreakRule[];
}

export interface TimeSegment {
  date: string;            // YYYY-MM-DD
  startTime: Date;
  endTime: Date;
  breakMinutes: number;
}

export interface OvertimeResult {
  byDay: DayResult[];
  totals: {
    regularMinutes: number;
    overtimeMinutes: number;
    doubletimeMinutes: number;
    breakMinutes: number;
    totalWorkedMinutes: number;
  };
  violations: ComplianceViolation[];
}

export function calculateOvertime(
  segments: TimeSegment[],
  payRule: PayRuleConfig,
  workweekStart: string    // YYYY-MM-DD of the workweek start
): OvertimeResult {
  const result: OvertimeResult = { byDay: [], totals: {...}, violations: [] };

  // Step 1: Calculate daily hours
  const dailyTotals = aggregateByDay(segments);

  // Step 2: Apply daily overtime (if configured, e.g., California)
  for (const day of dailyTotals) {
    const worked = day.workedMinutes;
    if (payRule.doubletimeDailyThresholdMinutes && worked > payRule.doubletimeDailyThresholdMinutes) {
      day.doubletimeMinutes = worked - payRule.doubletimeDailyThresholdMinutes;
      day.overtimeMinutes = payRule.doubletimeDailyThresholdMinutes - (payRule.overtimeDailyThresholdMinutes ?? worked);
      day.regularMinutes = payRule.overtimeDailyThresholdMinutes ?? worked;
    } else if (payRule.overtimeDailyThresholdMinutes && worked > payRule.overtimeDailyThresholdMinutes) {
      day.overtimeMinutes = worked - payRule.overtimeDailyThresholdMinutes;
      day.regularMinutes = payRule.overtimeDailyThresholdMinutes;
    } else {
      day.regularMinutes = worked;
    }
  }

  // Step 3: Apply weekly overtime (FLSA: anything over 40hrs/week is OT)
  // Weekly OT is calculated on hours not already classified as daily OT
  const weeklyRegular = dailyTotals.reduce((sum, d) => sum + d.regularMinutes, 0);
  if (weeklyRegular > payRule.overtimeWeeklyThresholdMinutes) {
    // Reclassify excess regular hours as weekly OT (FLSA)
    redistributeWeeklyOvertime(dailyTotals, payRule.overtimeWeeklyThresholdMinutes);
  }

  // Step 4: Check compliance violations
  if (payRule.maxWeeklyMinutes) {
    const totalWeekly = dailyTotals.reduce((sum, d) => sum + d.workedMinutes, 0);
    if (totalWeekly > payRule.maxWeeklyMinutes) {
      result.violations.push({
        type: 'max_weekly_hours_exceeded',
        regulation: 'EU WTD Art. 6',
        threshold: payRule.maxWeeklyMinutes,
        actual: totalWeekly,
      });
    }
  }

  // Step 5: Check daily rest violations (EU WTD)
  if (payRule.minDailyRestMinutes) {
    checkDailyRestViolations(segments, payRule.minDailyRestMinutes, result.violations);
  }

  return result;
}
```

**Testing:**
- **FLSA basic:** 45 hours in a week = 40 regular + 5 overtime at 1.5x
- **FLSA edge:** Exactly 40 hours = 40 regular, 0 overtime
- **FLSA edge:** 39 hours 59 minutes = all regular, 0 overtime
- **California daily:** 10 hours in one day = 8 regular + 2 overtime at 1.5x
- **California double-time:** 14 hours in one day = 8 regular + 4 OT (8-12hrs) + 2 DT (>12hrs) at 2.0x
- **California 7th day:** 7 consecutive days worked, 7th day = first 8 hours at 1.5x, remaining at 2.0x
- **California + FLSA combined:** 10hrs/day for 5 days = daily OT (2hrs/day * 5 = 10hrs OT) + no additional weekly OT (since daily OT already classified 10hrs as OT, weekly regular = 40hrs = threshold)
- **EU WTD max weekly:** 50 hours in a week = violation detected with 'max_weekly_hours_exceeded'
- **EU WTD daily rest:** Only 9 hours between end of one shift and start of next = violation detected with 'min_daily_rest_violated'
- **EU WTD weekly rest:** No 24-hour rest period in a 7-day span = violation detected
- **Break rule:** 6-hour shift with no 30-minute break = break violation flagged
- **Empty input:** No time segments = zero totals, no violations
- **Timezone handling:** Entries crossing midnight are correctly attributed to the calendar day they started on
- **Fractional minutes:** Engine handles minute-level precision without rounding errors

### Task 3.2: Pay Rule CRUD & Jurisdiction Seeding

**What:** Implement API endpoints for managing pay rules with jurisdiction-specific JSONB overrides. Seed the database with pre-built pay rules for the most common US states and EU countries.

**Design:**

```typescript
// apps/api/src/modules/pay-rule/pay-rule.routes.ts
// GET    /api/v1/pay-rules           — list all pay rules for tenant
// POST   /api/v1/pay-rules           — create custom pay rule
// GET    /api/v1/pay-rules/:id       — get pay rule with resolved JSONB rules
// PUT    /api/v1/pay-rules/:id       — update pay rule
// POST   /api/v1/pay-rules/preview   — preview overtime calculation with a pay rule

// Seed data covers:
// US Federal (FLSA), US-CA, US-NY, US-TX, US-IL, US-FL, US-WA, US-OR, US-CO, US-NV
// EU-DE, EU-FR, EU-GB (post-Brexit WTR), EU-ES, EU-NL
```

**Testing:**
- GET /api/v1/pay-rules returns seeded FLSA and California rules
- POST /api/v1/pay-rules creates a custom pay rule with JSONB jurisdiction overrides
- Preview endpoint calculates overtime for sample time entries against a given pay rule and returns the breakdown
- California pay rule returns correct JSONB: `{"overtime_daily_threshold": 480, "doubletime_daily_threshold": 720, ...}`
- EU Germany pay rule includes max_daily_hours, sunday_work_restricted fields in JSONB
- Pay rules can be assigned to employees via employee.complianceData.overtime_rule field
- Attempting to delete a pay rule assigned to employees returns 409 Conflict

### Task 3.3: Automatic Timesheet Generation

**What:** Implement a background process that automatically generates weekly timesheets from time entries. Timesheets aggregate daily hours, apply the employee's assigned pay rule, and compute regular/overtime/double-time breakdowns.

**Design:**

```typescript
// apps/api/src/modules/timesheet/timesheet-generator.ts
export class TimesheetGenerator {
  async generateForWeek(tenantId: string, weekStart: Date): Promise<Timesheet[]> {
    const employees = await this.employeeService.findActive(tenantId);
    const timesheets: Timesheet[] = [];

    for (const emp of employees) {
      // 1. Get all time entries for this employee in the week
      const entries = await this.timeEntryService.findByDateRange(
        emp.id, weekStart, addDays(weekStart, 7)
      );

      // 2. Convert entries to time segments (pair clock-ins with clock-outs)
      const segments = this.pairEntriesToSegments(entries);

      // 3. Get employee's pay rule
      const payRule = await this.payRuleService.getForEmployee(emp.id);

      // 4. Calculate overtime
      const result = calculateOvertime(segments, payRule, formatDate(weekStart));

      // 5. Create or update timesheet
      const timesheet = await this.upsertTimesheet(emp.id, tenantId, weekStart, result);
      timesheets.push(timesheet);
    }

    return timesheets;
  }
}
```

**Testing:**
- Timesheet is generated with correct regular and overtime minutes for a standard 45-hour FLSA week
- Timesheet hours_summary JSONB contains by_day breakdown with each day's hours
- Employee with California pay rule gets daily overtime calculated correctly in timesheet
- Missing clock-out (orphan clock-in) flags the timesheet for review
- Re-running generation for the same week updates existing timesheet (upsert, not duplicate)
- Timesheet status starts as 'draft'
- Timesheet correctly handles an employee who worked at multiple locations during the week

---

## Phase 4: Timesheet Approval Workflow

**Goal:** Implement the manager approval workflow for timesheets, including submission, approval, rejection, and locking for payroll export.

**Duration estimate:** 2-3 weeks

**Dependencies:** Phase 3 (timesheets must exist to approve)

### Task 4.1: Approval State Machine

**What:** Implement the timesheet state machine: draft -> submitted -> approved/rejected. Rejected timesheets return to draft. Approved timesheets can be locked for payroll. State transitions are enforced server-side.

**Design:**

```typescript
// apps/api/src/modules/timesheet/timesheet-state-machine.ts
const VALID_TRANSITIONS: Record<TimesheetStatus, TimesheetStatus[]> = {
  draft:     ['submitted'],
  submitted: ['approved', 'rejected'],
  approved:  ['locked', 'submitted'],  // can reopen
  rejected:  ['draft'],                 // employee corrects and resubmits
  locked:    [],                        // terminal state (payroll exported)
};

export class TimesheetStateMachine {
  async transition(
    timesheetId: string,
    targetStatus: TimesheetStatus,
    actor: { employeeId: string; roles: string[] },
    metadata?: { reason?: string }
  ): Promise<Timesheet> {
    const timesheet = await this.getTimesheet(timesheetId);

    // Validate transition
    if (!VALID_TRANSITIONS[timesheet.status].includes(targetStatus)) {
      throw new BadRequestError(
        `Cannot transition from '${timesheet.status}' to '${targetStatus}'`
      );
    }

    // Authorization checks
    switch (targetStatus) {
      case 'submitted':
        this.assertIsOwner(timesheet, actor);
        break;
      case 'approved':
      case 'rejected':
        await this.assertCanApprove(timesheet, actor);
        break;
      case 'locked':
        this.assertIsAdmin(actor);
        break;
    }

    // Apply transition
    const approval = timesheet.approval ?? {};
    if (targetStatus === 'approved') {
      approval.approvals = approval.approvals ?? [];
      approval.approvals.push({
        level: approval.approvals.length + 1,
        approvedBy: actor.employeeId,
        approvedAt: new Date().toISOString(),
        notes: metadata?.reason,
      });
    }
    if (targetStatus === 'rejected') {
      approval.rejection = {
        rejectedBy: actor.employeeId,
        rejectedAt: new Date().toISOString(),
        reason: metadata?.reason ?? 'No reason provided',
      };
    }

    const [updated] = await db.update(timesheet_table)
      .set({
        status: targetStatus,
        approval,
        updatedAt: new Date(),
      })
      .where(eq(timesheet_table.id, timesheetId))
      .returning();

    // Notify relevant parties
    await this.notifyTransition(updated, targetStatus, actor);

    return updated;
  }
}
```

**Testing:**
- Employee can submit their own draft timesheet (draft -> submitted)
- Employee cannot submit another employee's timesheet (403)
- Manager can approve a submitted timesheet for their direct report
- Manager cannot approve their own timesheet (conflict of interest check)
- Manager can reject a submitted timesheet with a mandatory reason
- Rejected timesheet returns to draft status for employee correction
- Employee cannot directly set status to approved (must go through submission)
- Admin can lock an approved timesheet for payroll
- Locked timesheet cannot be modified or transition to any other state
- Approval JSONB records the complete approval chain with timestamps
- Notification is sent to employee when their timesheet is approved or rejected
- Notification is sent to manager when an employee submits a timesheet

### Task 4.2: Timesheet Review UI

**What:** Build the manager timesheet review interface showing submitted timesheets with daily hour breakdowns, overtime calculations, flags, and approve/reject actions.

**Design:**

```tsx
// apps/web/src/app/(dashboard)/timesheets/review/page.tsx
// Manager view: list of submitted timesheets with:
// - Employee name, department, location
// - Week period (e.g., May 4 - May 10, 2026)
// - Total hours (regular / OT / DT) with visual bar chart
// - Flagged entries highlighted in amber
// - Approve button (green) / Reject button (red) with reason modal
// - Click-through to detailed daily view showing individual punches

// Daily detail view:
// | Day       | Clock In | Clock Out | Break | Worked | Regular | OT   | DT   | Flags |
// |-----------|----------|-----------|-------|--------|---------|------|------|-------|
// | Mon 05/04 | 7:58 AM  | 5:02 PM  | 30m   | 9h 4m  | 8h 0m   | 1h 4m| 0    |       |
// | Tue 05/05 | 7:55 AM  | 6:30 PM  | 30m   | 10h 5m | 8h 0m   | 2h 5m| 0    |       |
// | ...       |          |          |       |        |         |      |      |       |
// | TOTAL     |          |          | 2h30m | 47h20m | 40h 0m  | 7h20m| 0    |       |
```

**Testing:**
- Manager sees only timesheets from employees in their scope (location/department)
- Submitted timesheets are displayed with correct hour breakdowns
- Flagged time entries (geofence violations, missing clock-out) are visually highlighted
- Approve action transitions the timesheet and shows success toast
- Reject action requires a reason in a modal before submission
- After approval, the timesheet disappears from the review queue
- Pagination works correctly for managers with many direct reports
- Empty state displayed when no timesheets are pending review

### Task 4.3: Manager Time Entry Correction

**What:** Allow managers to adjust time entries (correct missed clock-outs, fix incorrect times) with mandatory audit trail. The adjusted_time field stores the correction while preserving the original punch_time.

**Design:**

```typescript
// Manager adjustment endpoint
// PATCH /api/v1/time-entries/:id/adjust
// Body: { adjusted_time: "2026-05-12T17:00:00-05:00", reason: "Employee forgot to clock out" }
```

**Testing:**
- Manager can adjust a time entry's time; original punch_time is preserved; adjusted_time is set
- Adjustment reason is mandatory; omitting it returns 400
- Audit record is created with old_value, new_value, changed_by, and change_reason
- Employee receives notification when their time entry is adjusted
- Manager cannot adjust time entries for employees outside their scope
- Adjusted time is used in overtime calculations (not the original punch_time)
- Multiple adjustments create multiple audit records (full history preserved)

---

## Phase 5: Employee Self-Service & Web UI

**Goal:** Build the employee-facing web interface: personal timesheet view, time-off request submission, profile management, and notification center.

**Duration estimate:** 3-4 weeks

**Dependencies:** Phase 4 (timesheets and approval workflow exist)

### Task 5.1: Employee Timesheet View

**What:** Employee-facing view of their own timesheets, showing daily hours, overtime breakdown, approval status, and submission action. Read-only for approved/locked timesheets.

**Design:**

```tsx
// apps/web/src/app/(employee)/timesheets/page.tsx
// - Week selector (previous/current/next week)
// - Daily breakdown table matching manager view format
// - Submission button when status is 'draft'
// - Status badge: Draft (gray), Submitted (blue), Approved (green), Rejected (red)
// - Rejection reason displayed when status is 'rejected'
// - Edit button for draft timesheets to add missing entries
// - Dispute button for approved timesheets (creates a correction request)
```

**Testing:**
- Employee sees only their own timesheets
- Current week timesheet shows accumulated hours even if incomplete
- Clicking "Submit" transitions draft to submitted and disables editing
- Rejected timesheet shows the rejection reason from the manager
- Employee can navigate to previous weeks using the week selector
- Approved timesheet shows green status badge and read-only view
- Overtime hours are visually distinguished from regular hours

### Task 5.2: Employee Profile & Settings

**What:** Self-service profile page where employees can update contact information, view their pay rule assignment, manage notification preferences, and see their role/permissions.

**Design:**

```tsx
// apps/web/src/app/(employee)/profile/page.tsx
// Sections:
// - Personal Information (name, email, phone) - editable
// - Employment Details (employee number, hire date, FLSA status) - read-only
// - Assigned Location & Department - read-only
// - Pay Rule Summary (overtime rules that apply to them) - read-only
// - Notification Preferences (email, push, in-app toggles) - editable
```

**Testing:**
- Employee can update their own email and phone number
- Employee cannot change their FLSA status, employee number, or hire date
- Pay rule summary displays the correct overtime rules for their jurisdiction
- Notification preferences are saved and respected by the notification system
- Profile changes are reflected immediately after save

### Task 5.3: Notification Center

**What:** In-app notification system displaying shift reminders, approval requests, compliance alerts, and time entry adjustments. Notifications are persisted and marked as read.

**Design:**

```typescript
// apps/api/src/modules/notification/notification.service.ts
export class NotificationService {
  async create(notification: CreateNotification): Promise<void> {
    const [record] = await db.insert(notificationTable).values({
      tenantId: notification.tenantId,
      recipientId: notification.recipientId,
      type: notification.type,
      content: {
        title: notification.title,
        body: notification.body,
        referenceType: notification.referenceType,
        referenceId: notification.referenceId,
        actionUrl: notification.actionUrl,
      },
      channel: notification.channel,
    }).returning();

    // Push to SSE stream for real-time delivery
    this.eventBus.emit('notification.created', record);

    // Queue email if channel includes email
    if (notification.channel === 'email') {
      await this.emailQueue.add('send', { notificationId: record.id });
    }
  }

  async getUnread(employeeId: string): Promise<Notification[]> {
    return db.select().from(notificationTable)
      .where(and(
        eq(notificationTable.recipientId, employeeId),
        isNull(notificationTable.readAt)
      ))
      .orderBy(desc(notificationTable.createdAt))
      .limit(50);
  }
}
```

**Testing:**
- Notification bell icon shows unread count
- Clicking a notification navigates to the relevant resource (timesheet, schedule, etc.)
- Marking a notification as read removes it from unread count
- "Mark all as read" clears all unread notifications
- Notifications arrive in real-time via SSE without page refresh
- Email notifications are sent when channel is set to 'email' in user preferences
- Notification types: timesheet_approved, timesheet_rejected, shift_reminder, compliance_alert, time_entry_adjusted

---

## Phase 6: Leave & Absence Management

**Goal:** Implement leave types, accrual calculations, leave request workflows, and FMLA case tracking.

**Duration estimate:** 3-4 weeks

**Dependencies:** Phase 1 (database schema; can run in parallel with Phases 2-4)

### Task 6.1: Leave Type Configuration

**What:** CRUD for leave types with jurisdiction-specific accrual rules stored in JSONB config. Support paid, unpaid, protected (FMLA), and statutory categories.

**Design:**

```typescript
// apps/api/src/modules/leave/leave-type.service.ts
// Leave types with JSONB config supporting:
// - Accrual method: per_pay_period, per_hour_worked, annual_grant, none
// - Accrual rate: hours per period or per hour worked
// - Max balance and carryover limits
// - Jurisdiction-specific overrides in config JSONB:
//   { "US-CA": { "sick_leave_rate": 0.0333, "max_balance": 48 } }
// - Protected status (cannot count toward attendance occurrences)
// - FMLA qualifying flag
```

**Testing:**
- Admin can create leave types: Vacation, Sick, FMLA, Bereavement, Jury Duty
- Leave types with accrual_method 'per_hour_worked' accumulate correctly (e.g., 1 hour per 30 hours worked)
- Carryover limits are enforced at year-end (excess balance is forfeited)
- FMLA leave type is flagged as protected and FMLA-qualifying
- Jurisdiction-specific overrides in JSONB are applied when the employee's jurisdiction matches
- Leave types can be deactivated without deleting (is_active = false)

### Task 6.2: Leave Request Workflow

**What:** Employee leave request submission with balance validation, manager approval workflow, and automatic balance deduction on approval.

**Design:**

```typescript
// apps/api/src/modules/leave/leave-request.service.ts
export class LeaveRequestService {
  async submit(tenantId: string, employeeId: string, data: CreateLeaveRequest): Promise<LeaveRequest> {
    // 1. Validate balance
    const balance = await this.getBalance(employeeId, data.leaveTypeId);
    if (balance.balanceHours < data.totalHours) {
      throw new BadRequestError(`Insufficient balance: ${balance.balanceHours}h available, ${data.totalHours}h requested`);
    }

    // 2. Check for conflicting leave requests
    const conflicts = await this.checkConflicts(employeeId, data.startDate, data.endDate);
    if (conflicts.length > 0) {
      throw new ConflictError('Overlapping leave request exists');
    }

    // 3. Create request
    const [request] = await db.insert(leaveRequest).values({
      tenantId,
      employeeId,
      leaveTypeId: data.leaveTypeId,
      startDate: data.startDate,
      endDate: data.endDate,
      totalHours: data.totalHours,
      status: 'pending',
      metadata: {
        reason: data.reason,
        halfDayStart: data.halfDayStart ?? false,
        halfDayEnd: data.halfDayEnd ?? false,
      },
    }).returning();

    // 4. Notify manager
    await this.notificationService.create({
      recipientId: (await this.getManager(employeeId)).id,
      type: 'leave_approval_needed',
      title: 'Leave Request Pending',
      body: `${employee.firstName} ${employee.lastName} requested ${data.totalHours}h of ${leaveType.name}`,
      referenceType: 'leave_request',
      referenceId: request.id,
    });

    return request;
  }

  async approve(requestId: string, approverId: string): Promise<LeaveRequest> {
    // Deduct balance atomically with approval
    return db.transaction(async (tx) => {
      const [req] = await tx.update(leaveRequest)
        .set({ status: 'approved', metadata: sql`metadata || ${JSON.stringify({ approval_chain: [{ approvedBy: approverId, approvedAt: new Date().toISOString() }] })}` })
        .where(eq(leaveRequest.id, requestId))
        .returning();

      await tx.update(leaveBalance)
        .set({
          balanceHours: sql`balance_hours - ${req.totalHours}`,
          usedYtd: sql`used_ytd + ${req.totalHours}`,
        })
        .where(and(
          eq(leaveBalance.employeeId, req.employeeId),
          eq(leaveBalance.leaveTypeId, req.leaveTypeId),
        ));

      return req;
    });
  }
}
```

**Testing:**
- Employee can submit a leave request with sufficient balance
- Request with insufficient balance returns 400 with remaining balance info
- Overlapping leave request returns 409 Conflict
- Manager approves; balance is deducted atomically
- Manager rejects; balance is not affected
- Employee cancels a pending request; status changes to cancelled
- Half-day requests correctly deduct half of a standard day's hours
- Notification sent to manager on submission, to employee on approval/rejection

### Task 6.3: Leave Balance Accrual

**What:** Background job that runs per pay period to calculate and credit leave accruals based on leave type configuration and hours worked.

**Design:**

```typescript
// Cron job: runs at end of each pay period
// For each active employee:
//   For each leave type with accrual_method != 'none':
//     If per_pay_period: credit accrual_rate hours
//     If per_hour_worked: credit (hours_worked * accrual_rate)
//     Cap at max_balance
//     Record adjustment in leave_balance.adjustments JSONB
```

**Testing:**
- Per-pay-period accrual credits the correct hours on schedule
- Per-hour-worked accrual calculates correctly from timesheet hours
- Balance does not exceed max_balance (excess is forfeited)
- New employees hired mid-period receive prorated accrual
- Terminated employees stop accruing
- Year-end carryover respects carryover_limit (excess balance forfeited)
- Adjustment JSONB records each accrual with date, amount, and method

### Task 6.4: FMLA Case Tracking

**What:** Implement FMLA case management: case creation, intermittent leave tracking, entitlement balance (480 hours / 12 weeks), and medical certification tracking.

**Design:**

```typescript
// FMLA cases are tracked in leave_request.metadata.fmla_case_id
// A separate FMLA tracking view shows:
// - Total entitlement: 480 hours (12 weeks * 40 hours)
// - Used hours (sum of approved FMLA leave requests)
// - Remaining hours
// - Intermittent leave pattern tracking
// - Medical certification status (received / pending / expired)
// - Case status: active, exhausted, closed
```

**Testing:**
- Creating an FMLA case sets total entitlement to 480 hours by default
- Leave requests linked to an FMLA case deduct from the FMLA entitlement
- Intermittent FMLA: multiple small leave requests correctly accumulate against the 480-hour total
- When FMLA hours are exhausted, the case status changes to 'exhausted'
- Medical certification status can be updated by HR/admin
- FMLA leave is coded as 'protected' and excluded from attendance occurrence tracking
- FMLA eligibility check: employee must have 12+ months tenure and 1,250+ hours worked in prior 12 months

---

## Phase 7: Scheduling Engine

**Goal:** Implement shift scheduling with availability management, conflict detection, coverage rules, shift swap workflows, and iCalendar export.

**Duration estimate:** 4-5 weeks

**Dependencies:** Phase 6 (leave requests must be visible to scheduling), Phase 2 (time entries for overtime constraint checking)

### Task 7.1: Schedule & Shift CRUD

**What:** Implement schedule periods and individual shift assignments. Schedules cover a date range for a location and contain multiple shifts. Shifts can be open (unassigned) or assigned to an employee.

**Design:**

```typescript
// apps/api/src/modules/scheduling/schedule.service.ts
// POST /api/v1/schedules         — create schedule period
// POST /api/v1/schedules/:id/shifts  — add shift to schedule
// POST /api/v1/schedules/:id/publish — publish schedule (notify employees)
// GET  /api/v1/schedules/my-shifts   — employee view of their shifts

// Shift creation with conflict detection:
export class ShiftService {
  async createShift(scheduleId: string, data: CreateShift): Promise<Shift> {
    if (data.employeeId) {
      // Check for shift conflicts
      const conflicts = await this.findConflicts(data.employeeId, data.startTime, data.endTime);
      if (conflicts.length > 0) {
        throw new ConflictError('Employee has a conflicting shift');
      }

      // Check for approved leave during shift
      const leave = await this.leaveService.findApprovedLeave(
        data.employeeId, data.shiftDate
      );
      if (leave) {
        throw new ConflictError('Employee has approved leave on this date');
      }

      // Check overtime threshold
      const weeklyHours = await this.getWeeklyHours(data.employeeId, data.shiftDate);
      const shiftMinutes = differenceInMinutes(data.endTime, data.startTime) - data.breakMinutes;
      const payRule = await this.payRuleService.getForEmployee(data.employeeId);
      if (weeklyHours + shiftMinutes > payRule.overtimeWeeklyThresholdMinutes) {
        // Warning, not blocking — attach metadata
        data.properties = {
          ...data.properties,
          overtimeWarning: true,
          projectedWeeklyMinutes: weeklyHours + shiftMinutes,
        };
      }
    }

    return db.insert(shift).values({ scheduleId, ...data }).returning();
  }
}
```

**Testing:**
- Create a schedule for a location and date range; verify it has 'draft' status
- Add shifts to a schedule; verify shifts are associated with the correct schedule
- Assigning an employee to a shift when they have a conflicting shift returns 409
- Assigning an employee on approved leave day returns 409
- Assigning a shift that would push the employee over 40 weekly hours adds an overtime warning but does not block
- Open shifts (no employee_id) can be created for future filling
- Publishing a schedule sends notifications to all assigned employees
- Published schedule is visible in employee's "My Shifts" view

### Task 7.2: Employee Availability

**What:** Allow employees to set their weekly availability windows and preferences. Scheduling respects availability constraints.

**Design:**

```typescript
// PUT /api/v1/employees/:id/availability
// Body: JSONB availability object with day-of-week windows and preferences
// Scheduling engine checks availability before shift assignment
```

**Testing:**
- Employee can set availability: "Mon-Fri 6am-10pm, not available Sat-Sun"
- Assigning a shift outside availability windows returns a warning (configurable: warning or block)
- Availability with effective_from/effective_to allows scheduling future availability changes
- Preferences (max hours/week, prefer morning) are stored and accessible to scheduling engine

### Task 7.3: Shift Swap Workflow

**What:** Allow employees to request shift swaps. Another employee can accept the swap. Manager approval is configurable (required by default). Compliance checks run on the swap to ensure no violations.

**Design:**

```typescript
// POST /api/v1/shifts/:id/swap-request     — employee offers shift for swap
// GET  /api/v1/swap-requests/available      — view available swap offers
// POST /api/v1/swap-requests/:id/accept     — another employee accepts
// POST /api/v1/swap-requests/:id/approve    — manager approves the swap
// The swap updates the shift's employee_id and creates audit records
```

**Testing:**
- Employee can create a swap request for their assigned shift
- Other eligible employees at the same location see the swap offer
- Accepting a swap triggers manager notification if approval required
- Manager approves; shift employee_id updates; both employees notified
- Manager rejects; shift remains unchanged; both employees notified
- Swap is blocked if accepting employee lacks required skills for the shift
- Swap is blocked if it would push accepting employee over overtime threshold
- Swap request expires after configurable period if not accepted

### Task 7.4: iCalendar Export

**What:** Generate RFC 5545 iCalendar feeds for employee schedules. Employees can subscribe to their schedule in Google Calendar, Apple Calendar, or Outlook.

**Design:**

```typescript
// packages/ical-export/src/index.ts
import { createEvents, EventAttributes } from 'ics';

export function shiftsToICalendar(shifts: Shift[], employeeName: string): string {
  const events: EventAttributes[] = shifts.map(shift => ({
    uid: shift.properties?.ical_uid ?? `shift-${shift.id}@timeattendance.example`,
    start: dateToArray(shift.startTime),
    end: dateToArray(shift.endTime),
    title: `Shift at ${shift.locationName}`,
    description: shift.notes ?? '',
    location: shift.locationName,
    categories: [shift.properties?.shift_type ?? 'regular'],
    status: 'CONFIRMED',
  }));

  const { value } = createEvents(events);
  return value;
}

// GET /api/v1/employees/:id/schedule.ics  — iCalendar feed URL
// Authentication via personal token in query parameter for calendar app compatibility
```

**Testing:**
- GET /schedule.ics returns valid iCalendar data (validate with ical-validator)
- Each shift appears as a VEVENT with correct DTSTART, DTEND, SUMMARY, and LOCATION
- Calendar subscription URL works in Google Calendar (add by URL)
- Calendar subscription URL works in Apple Calendar
- Updated shifts appear in the calendar after the app refreshes the subscription
- Cancelled shifts appear as CANCELLED events
- Feed only contains future shifts (past shifts excluded after 30 days)

---

## Phase 8: Compliance Engine & Alerting

**Goal:** Build the real-time compliance monitoring engine that detects approaching and actual violations for FLSA, EU WTD, and configurable jurisdiction rules. Generate proactive alerts to managers.

**Duration estimate:** 3-4 weeks

**Dependencies:** Phase 3 (overtime engine), Phase 2 (time entries as input to compliance checks)

### Task 8.1: Compliance Rule Configuration

**What:** Implement the compliance_config table with JSONB rules. Pre-seed rules for FLSA overtime threshold warnings, EU WTD max weekly hours, California meal break requirements, and configurable attendance occurrence policies.

**Design:**

```typescript
// Compliance rules are evaluated on every time entry event
// Rules are stored in compliance_config.rules JSONB array:
// [
//   { rule_type: "approaching_overtime", threshold_minutes: 2280, severity: "warning" },
//   { rule_type: "max_weekly_hours", threshold_minutes: 2880, severity: "violation" },
//   { rule_type: "missed_break", after_minutes: 300, required_break_minutes: 30, severity: "warning" }
// ]
```

**Testing:**
- Admin can configure compliance rules per tenant
- Pre-seeded FLSA rule: warn at 38 hours (2280 minutes) approaching 40-hour threshold
- Pre-seeded EU WTD rule: violation at 48 hours (2880 minutes)
- California meal break rule: warning if no 30-minute break after 5 hours
- Rules can be activated/deactivated without deletion
- Custom rules can reference any regulation in the regulation_reference field

### Task 8.2: Real-Time Compliance Evaluation

**What:** On every TimeEntryRecorded event, evaluate all active compliance rules for the affected employee. Generate compliance alerts for approaching thresholds and actual violations.

**Design:**

```typescript
// apps/api/src/modules/compliance/compliance-evaluator.ts
export class ComplianceEvaluator {
  async evaluate(timeEntry: TimeEntry): Promise<ComplianceAlert[]> {
    const employee = await this.employeeService.findById(timeEntry.employeeId);
    const rules = await this.getRulesForEmployee(employee);
    const alerts: ComplianceAlert[] = [];

    for (const rule of rules) {
      switch (rule.rule_type) {
        case 'approaching_overtime': {
          const weeklyMinutes = await this.getWeeklyMinutes(employee.id, timeEntry.punchTime);
          if (weeklyMinutes >= rule.threshold_minutes) {
            alerts.push({
              tenantId: employee.tenantId,
              employeeId: employee.id,
              alertData: {
                ruleType: 'approaching_overtime',
                regulation: 'FLSA',
                severity: rule.severity,
                currentWeeklyMinutes: weeklyMinutes,
                thresholdMinutes: rule.threshold_minutes,
                remainingMinutes: 2400 - weeklyMinutes, // OT threshold - current
              },
              status: 'open',
            });
          }
          break;
        }
        case 'missed_break': {
          const continuousMinutes = await this.getContinuousWorkedMinutes(employee.id, timeEntry.punchTime);
          if (continuousMinutes >= rule.after_minutes) {
            const hasBreak = await this.hasBreakInWindow(employee.id, timeEntry.punchTime, rule.after_minutes);
            if (!hasBreak) {
              alerts.push({ /* missed break alert */ });
            }
          }
          break;
        }
        // Additional rule types...
      }
    }

    // Persist alerts and notify
    for (const alert of alerts) {
      await this.persistAndNotify(alert);
    }

    return alerts;
  }
}
```

**Testing:**
- Employee clocks in for their 39th hour: "approaching_overtime" warning generated
- Employee clocks in for their 41st hour: overtime is calculated, no duplicate warning
- Employee works 49 hours in EU WTD jurisdiction: "max_weekly_hours" violation generated
- Employee works 6 hours without a break in California: "missed_break" warning generated
- Duplicate alerts are not generated for the same violation in the same period
- Alerts include context data (current hours, threshold, remaining)
- Manager receives notification for each new compliance alert
- Alert status lifecycle: open -> acknowledged -> resolved

### Task 8.3: Compliance Dashboard

**What:** Manager-facing dashboard showing all open compliance alerts with severity indicators, affected employees, and resolution actions.

**Design:**

```tsx
// apps/web/src/app/(dashboard)/compliance/page.tsx
// Dashboard displays:
// - Summary cards: X warnings, Y violations, Z unresolved
// - Alert list with severity badges (yellow warning, red violation)
// - Each alert shows: employee name, rule type, regulation reference, context
// - Acknowledge button (manager saw it)
// - Resolve button with resolution notes
// - Filters: by severity, by rule type, by date range
```

**Testing:**
- Dashboard shows correct count of open warnings and violations
- Violation alerts appear with red severity badge
- Warning alerts appear with amber severity badge
- Clicking acknowledge updates status and records who acknowledged
- Clicking resolve requires resolution notes and updates status
- Resolved alerts move to a "Resolved" tab
- Filters correctly narrow the displayed alerts
- Dashboard auto-refreshes via SSE when new alerts are generated

### Task 8.4: Audit Event Log

**What:** Implement the append-only audit_event table for compliance-critical operations. This is a simplified version of event sourcing (from Data Model Suggestion 2) applied specifically to audit-relevant actions.

**Design:**

```sql
-- Append-only audit log for compliance
CREATE TABLE audit_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    actor_id        UUID NOT NULL,
    actor_type      VARCHAR(20) NOT NULL,    -- employee, manager, system, api
    resource_type   VARCHAR(50) NOT NULL,     -- time_entry, timesheet, leave_request, etc.
    resource_id     UUID NOT NULL,
    payload         JSONB NOT NULL,           -- event-specific data
    ip_address      INET,
    user_agent      TEXT,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- This table is INSERT-only. No UPDATE or DELETE operations are permitted.
-- Application-level enforcement + database trigger to prevent modifications.
```

**Testing:**
- Every time entry creation, adjustment, and deletion creates an audit event
- Every timesheet submission, approval, and rejection creates an audit event
- Every compliance alert acknowledgment and resolution creates an audit event
- Audit events cannot be modified or deleted via the API
- Audit events include actor identity, IP address, and timestamp
- Audit log query supports date range, resource type, and actor filters
- DCAA-relevant events include project_code and labor_category in payload

---

## Phase 9: Payroll Export & Integrations

**Goal:** Implement payroll data export in multiple formats (CSV, HR-JSON Timecard, QuickBooks, Xero) and webhook-based integration infrastructure.

**Duration estimate:** 2-3 weeks

**Dependencies:** Phase 4 (approved/locked timesheets as export source)

### Task 9.1: Payroll Export Engine

**What:** Build the `packages/payroll-formats` library that transforms approved timesheet data into standard payroll export formats. Implement CSV, HR-JSON Timecard (HR Open Standards), QuickBooks IIF, and Xero CSV formats.

**Design:**

```typescript
// packages/payroll-formats/src/index.ts
export interface PayrollRecord {
  employeeNumber: string;
  employeeName: string;
  periodStart: string;
  periodEnd: string;
  regularHours: number;
  overtimeHours: number;
  doubletimeHours: number;
  ptoHours: number;
  holidayHours: number;
  payCodes: PayCode[];
}

export function formatCSV(records: PayrollRecord[]): string { /* ... */ }
export function formatHRJsonTimecard(records: PayrollRecord[]): object { /* ... */ }
export function formatQuickBooksIIF(records: PayrollRecord[]): string { /* ... */ }
export function formatXeroCSV(records: PayrollRecord[]): string { /* ... */ }

// API endpoint
// POST /api/v1/payroll-exports
// Body: { period_start: "2026-05-04", period_end: "2026-05-10", format: "csv" }
// Response: { id: "uuid", status: "completed", download_url: "/api/v1/payroll-exports/:id/download" }
```

**Testing:**
- CSV export contains correct headers and one row per employee with regular/OT/DT hours
- HR-JSON Timecard output validates against the HR Open Standards schema
- QuickBooks IIF format is importable into QuickBooks Desktop
- Xero CSV format is importable into Xero payroll
- Export only includes timesheets with 'approved' or 'locked' status
- Export generates a downloadable file with correct MIME type
- Export records which timesheets were included (payroll_export table)
- Timesheets are locked after successful export (preventing modification)
- Shift differentials and pay codes are correctly included in export records

### Task 9.2: Webhook Infrastructure

**What:** Implement outbound webhook subscriptions so external systems can receive real-time events (timesheet approved, compliance violation, shift published).

**Design:**

```typescript
// apps/api/src/modules/webhook/webhook.service.ts
export class WebhookService {
  async deliver(tenantId: string, eventType: string, payload: unknown): Promise<void> {
    const subscriptions = await db.select().from(webhookSubscription)
      .where(and(
        eq(webhookSubscription.tenantId, tenantId),
        eq(webhookSubscription.isActive, true),
        sql`${eventType} = ANY(${webhookSubscription.eventTypes})`
      ));

    for (const sub of subscriptions) {
      const signature = createHmac('sha256', sub.secretHash)
        .update(JSON.stringify(payload))
        .digest('hex');

      await this.deliveryQueue.add('webhook-delivery', {
        url: sub.url,
        payload,
        headers: {
          'Content-Type': 'application/json',
          'X-Webhook-Signature': signature,
          'X-Event-Type': eventType,
        },
        subscriptionId: sub.id,
        retryCount: 0,
      });
    }
  }
}
```

**Testing:**
- Admin can create a webhook subscription for specific event types
- Webhook payload is signed with HMAC-SHA256 using the subscription secret
- Delivery retries up to 3 times with exponential backoff on failure
- After 3 consecutive failures, subscription is marked inactive
- Webhook delivery log shows status, response code, and latency for each attempt
- All documented event types trigger webhook delivery: timesheet.approved, timesheet.rejected, compliance.violation, schedule.published, shift.swapped

### Task 9.3: OAuth 2.0 API Client Management

**What:** Implement OAuth 2.0 client credentials flow for third-party API consumers (payroll systems, BI tools, HRIS platforms).

**Design:**

```typescript
// POST /api/v1/oauth/clients    — register API client
// POST /api/v1/oauth/token      — exchange client_id + client_secret for bearer token
// Token includes tenant_id and scoped permissions
// Token expiry: 1 hour; refresh via client credentials (no refresh token)
```

**Testing:**
- Admin can register an OAuth client with client_id and client_secret
- POST /oauth/token with valid credentials returns a bearer token
- Bearer token includes tenant_id and scoped permissions
- Expired tokens (>1 hour) return 401
- Invalid client credentials return 401
- Rate limiting is applied per client_id (configurable, default 1000 req/hour)
- Token scopes restrict access: a 'read' scope client cannot POST time entries

---

## Phase 10: Biometric Clock-In & Data Sovereignty

**Goal:** Implement on-device facial recognition for clock-in verification using open-source embedding models. All biometric data is processed and stored locally -- never transmitted to external services.

**Duration estimate:** 4-5 weeks

**Dependencies:** Phase 2 (clock-in infrastructure)

### Task 10.1: Facial Recognition SDK

**What:** Build the `packages/biometric-client` library that performs facial recognition on commodity Android/iOS devices using FaceNet or DeepFace (open-source models). The SDK captures a face image, generates an embedding vector, and matches against stored templates -- all on-device.

**Design:**

```typescript
// packages/biometric-client/src/index.ts
// Uses TensorFlow.js or ONNX Runtime for on-device inference
// Model: FaceNet (InceptionResNetV1) or ArcFace (ResNet-100)
// Embedding dimension: 128 or 512 floats
// Matching: cosine similarity with configurable threshold (default 0.7)

export interface BiometricResult {
  matched: boolean;
  confidence: number;        // 0.0 - 1.0
  livenessCheck: boolean;    // anti-spoofing (blink detection, head turn)
  templateId?: string;       // matched template UUID
}

export class FacialRecognitionClient {
  async enroll(employeeId: string, imageData: Uint8Array): Promise<BiometricTemplate> {
    // 1. Detect face in image (MTCNN or BlazeFace)
    // 2. Align and crop face
    // 3. Generate embedding vector (FaceNet inference)
    // 4. Encrypt embedding with employee-specific key
    // 5. Store encrypted template in biometric_template table
    // 6. Return template metadata (algorithm, quality score)
  }

  async verify(employeeId: string, imageData: Uint8Array): Promise<BiometricResult> {
    // 1. Detect and align face
    // 2. Liveness check (anti-spoofing)
    // 3. Generate embedding vector
    // 4. Load stored templates for employee
    // 5. Decrypt templates and compute cosine similarity
    // 6. Return match result with confidence score
  }
}
```

**Testing:**
- Enrollment captures face image and generates encrypted biometric template
- Verification matches the correct employee with confidence >= 0.7
- Verification rejects a different person (confidence < threshold)
- Liveness check detects a photo-of-a-photo attack (returns livenessCheck: false)
- No biometric data is transmitted to any external API (network monitoring confirms)
- Template encryption uses AES-256 with per-employee keys
- Algorithm name and version are stored in template metadata for auditability
- ISO 19795 quality score is calculated and stored with each template

### Task 10.2: Biometric Consent Management

**What:** Implement the consent workflow required by BIPA, GDPR, and CCPA. Employees must explicitly consent before biometric enrollment. Consent records are immutable and include the exact consent text, timestamp, method, and applicable jurisdiction law.

**Design:**

```typescript
// Consent flow:
// 1. Employee is shown consent text specific to their jurisdiction (BIPA, GDPR, CCPA)
// 2. Employee signs electronically (tap/click "I Agree")
// 3. Consent record is created with full text, jurisdiction_law, and timestamp
// 4. Only after consent is granted can biometric enrollment proceed
// 5. Employee can revoke consent at any time; templates are destroyed on revocation

// Consent text is jurisdiction-specific:
// BIPA: includes specific disclosures required by 740 ILCS 14/15(b)
// GDPR: includes Article 9 lawful basis and data retention period
// CCPA: includes right to opt-out and data access rights
```

**Testing:**
- Consent prompt is shown before biometric enrollment
- Consent text varies by employee jurisdiction (BIPA vs GDPR vs CCPA)
- Consent record includes exact text, IP address, user agent, and timestamp
- Attempting enrollment without active consent returns 403
- Revoking consent triggers immediate destruction of all biometric templates
- Destroyed templates are overwritten (not just soft-deleted) per BIPA requirement
- Consent records are immutable; revocation is recorded alongside the original grant
- Scheduled destruction dates are set based on the retention period in the consent

### Task 10.3: Biometric Data Lifecycle

**What:** Implement the full lifecycle: enrollment, verification on clock-in, template rotation, and scheduled destruction. Ensure GDPR right-to-erasure compliance.

**Design:**

```typescript
// Scheduled destruction job:
// - Runs daily
// - Finds templates where scheduled_destruction_date <= today AND destroyed_at IS NULL
// - Overwrites template_data with zeros
// - Sets destroyed_at timestamp
// - Logs destruction in audit_event table

// GDPR erasure:
// - On employee termination, schedule template destruction per retention period
// - On explicit erasure request, destroy templates immediately
// - Audit event records the destruction for compliance proof
```

**Testing:**
- Templates past their destruction date are automatically destroyed
- Destroyed templates cannot be used for verification
- Employee termination triggers scheduled destruction per retention policy
- GDPR erasure request destroys templates immediately (not waiting for scheduled date)
- Destruction audit event is created with template ID, destruction timestamp, and trigger reason
- Re-enrollment after destruction requires new consent

---

## Phase 11: AI-Powered Scheduling & NLP

**Goal:** Implement AI-driven schedule optimization and natural language schedule management for managers.

**Duration estimate:** 4-5 weeks

**Dependencies:** Phase 7 (scheduling engine with shifts, availability, and constraints)

### Task 11.1: AI Schedule Optimization

**What:** Build an AI-powered auto-scheduling engine that generates optimal shift assignments considering demand forecasting, employee availability, skills requirements, overtime constraints, and employee preferences.

**Design:**

```typescript
// apps/api/src/modules/ai/schedule-optimizer.ts
export class ScheduleOptimizer {
  async generateSchedule(
    locationId: string,
    periodStart: Date,
    periodEnd: Date,
    constraints: ScheduleConstraints
  ): Promise<ProposedSchedule> {
    // 1. Gather inputs
    const employees = await this.getEligibleEmployees(locationId);
    const availability = await this.getAvailability(employees, periodStart, periodEnd);
    const skills = await this.getEmployeeSkills(employees);
    const currentHours = await this.getCurrentWeeklyHours(employees);
    const payRules = await this.getPayRules(employees);
    const historicalDemand = await this.getDemandForecast(locationId, periodStart, periodEnd);

    // 2. Formulate as constraint satisfaction problem
    // Constraints: availability, skills, overtime limits, rest periods, preferences
    // Objective: minimize labor cost while meeting coverage targets

    // 3. Use Claude API for complex constraint reasoning
    const prompt = this.buildSchedulingPrompt({
      employees, availability, skills, currentHours,
      payRules, historicalDemand, constraints,
    });

    const response = await this.claude.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 4096,
      messages: [{ role: 'user', content: prompt }],
    });

    // 4. Parse and validate proposed assignments
    const proposed = this.parseScheduleResponse(response);
    const validated = await this.validateProposal(proposed);

    return validated;
  }
}
```

**Testing:**
- Auto-scheduler generates a valid schedule that covers all required shifts
- No employee is scheduled outside their availability windows
- No employee is scheduled for a shift requiring skills they lack
- No employee is pushed over their overtime threshold
- EU WTD rest periods (11h daily, 24h weekly) are respected
- Schedule meets minimum coverage targets per shift
- Optimization prefers employee preferences when multiple valid assignments exist
- Schedule generation completes within 30 seconds for 100 employees / 1 week
- Proposed schedule can be reviewed and modified before publishing

### Task 11.2: Natural Language Schedule Requests

**What:** Enable managers to create and modify schedules using natural language. "Schedule 3 certified welders for the night shift next Tuesday, avoid overtime" should produce the correct shift assignments.

**Design:**

```typescript
// POST /api/v1/schedules/natural-language
// Body: { request: "Schedule 3 certified welders for the night shift next Tuesday, avoid overtime" }
// Response: { proposed_changes: [...], explanation: "..." }

export class NLPScheduleHandler {
  async processRequest(
    tenantId: string,
    managerId: string,
    request: string
  ): Promise<ScheduleProposal> {
    // 1. Use Claude to parse intent
    const parsed = await this.parseSchedulingIntent(request);
    // Parsed: { action: "schedule", count: 3, skill: "certified_welder",
    //           shift_type: "night", date: "2026-05-26", constraint: "avoid_overtime" }

    // 2. Find eligible employees
    const eligible = await this.findEligible(parsed);

    // 3. Generate proposal respecting constraints
    const proposal = await this.optimizer.proposeForRequest(parsed, eligible);

    // 4. Return proposal for manager confirmation
    return proposal;
  }
}
```

**Testing:**
- "Schedule 3 welders for Tuesday night shift" correctly identifies: count=3, skill=welder, date=next Tuesday, shift=night
- "Move Alice from Monday to Wednesday" correctly identifies a shift reassignment
- "Who is available for overtime this weekend?" returns eligible employees under the OT threshold
- "Cancel all shifts for the holiday" identifies the holiday date and creates cancellation proposals
- Invalid requests ("Schedule a unicorn") return a helpful error message
- Proposals require manager confirmation before being applied
- Ambiguous requests ("next week") prompt for clarification

### Task 11.3: Anomaly Detection

**What:** Implement local ML-based anomaly detection for buddy-punching patterns, unusual clock-in times, geofencing violations, and attendance irregularities.

**Design:**

```typescript
// Local inference using TensorFlow.js or ONNX Runtime
// Models trained on historical attendance data per tenant
// Anomaly types:
// - Buddy punching: two employees always clock in within 30 seconds of each other at same kiosk
// - Time pattern anomaly: employee suddenly changes clock-in pattern (3 weeks of 7am, then 11am)
// - Geofence anomaly: frequent just-inside-geofence punches (gaming the boundary)
// - Absence prediction: historical pattern suggests high absence probability for upcoming dates
```

**Testing:**
- Buddy-punch detection flags two employees who clock in within 30 seconds at the same location consistently (>5 occurrences)
- Time pattern anomaly detects a significant deviation from an employee's normal clock-in time
- Geofence anomaly detects clusters of punches near the geofence boundary
- False positive rate is below 5% on the test dataset
- Anomaly detection runs asynchronously (does not block clock-in flow)
- Detection results are stored as compliance alerts with 'anomaly' type
- Models can be retrained on updated data without system downtime

### Task 11.4: MCP Server

**What:** Expose the T&A system's core capabilities via the Model Context Protocol (MCP) so AI assistants (Claude, Copilot) can directly query schedules, approve timesheets, and surface compliance alerts.

**Design:**

```typescript
// apps/api/src/modules/ai/mcp-server.ts
// MCP tools exposed:
// - get_employee_schedule(employee_id, date_range)
// - get_pending_timesheets(manager_id)
// - approve_timesheet(timesheet_id, notes)
// - get_compliance_alerts(status, severity)
// - get_attendance_summary(employee_id, date_range)
// - record_clock_in(employee_id, source)
// MCP resources:
// - /schedules/{location_id}/{week}
// - /timesheets/{employee_id}/{period}
// - /compliance/alerts/{status}
```

**Testing:**
- MCP server starts and registers tools with the MCP protocol
- AI assistant can query "Who is clocked in right now?" via get_attendance_summary
- AI assistant can approve a timesheet via approve_timesheet (with auth)
- AI assistant receives compliance alerts via get_compliance_alerts
- Tool responses follow MCP schema conventions
- Authentication is enforced on all MCP tool calls

---

## Phase 12: Mobile App & Kiosk Mode

**Goal:** Build React Native mobile app for employee clock-in (with GPS and camera) and a dedicated kiosk mode for shared tablet devices.

**Duration estimate:** 4-5 weeks

**Dependencies:** Phase 2 (clock-in API), Phase 5 (employee self-service patterns), Phase 10 (biometric SDK)

### Task 12.1: React Native Mobile App

**What:** Build the employee-facing mobile app with clock-in/out, schedule viewing, leave requests, and push notifications. GPS geolocation is captured automatically on clock events.

**Design:**

```typescript
// apps/mobile/ (React Native with Expo)
// Screens:
// - Clock In/Out (primary screen with large button, GPS auto-capture)
// - My Schedule (weekly view of assigned shifts)
// - Timesheets (view and submit weekly timesheets)
// - Leave Requests (submit and track leave)
// - Notifications (push notification history)
// - Profile (view profile and settings)
```

**Testing:**
- App installs on iOS 16+ and Android 13+
- Clock-in captures GPS coordinates and submits to API
- Clock-in works offline (queued and synced when connectivity returns)
- Push notifications arrive for shift reminders, approval results, and compliance alerts
- Schedule view displays assigned shifts with correct times in employee's local timezone
- Leave request submission works with date picker and leave type selector
- Biometric login (Face ID / Touch ID) for app unlock
- App performance: clock-in screen loads in <2 seconds on mid-range device

### Task 12.2: Kiosk Mode

**What:** Build a dedicated kiosk mode for shared tablet devices placed at work locations. Employees clock in by facial recognition or employee number entry. The kiosk runs as a full-screen app locked to the clock-in screen.

**Design:**

```typescript
// Kiosk mode runs as a separate Next.js route or React Native config
// Features:
// - Full-screen, auto-locked interface (no browser chrome, no navigation)
// - Camera-based facial recognition for hands-free clock-in
// - Fallback: employee number entry on a numeric keypad
// - Location is fixed (configured for the device's installed location)
// - Automatic session timeout (returns to clock-in screen after 30 seconds)
// - Admin PIN to exit kiosk mode or configure settings
// - Offline queue: clock events stored locally if network is down

// Security:
// - Kiosk device authenticates with a device-specific API token
// - No employee credentials are stored on the kiosk
// - Camera images are processed on-device; only embeddings are transmitted
```

**Testing:**
- Kiosk displays a camera feed for facial recognition
- Recognized employee sees "Welcome, [Name]" and is clocked in automatically
- Unrecognized face prompts for employee number entry
- Employee number entry on numeric keypad works as fallback
- Kiosk returns to idle screen after 30 seconds of inactivity
- Clock events are queued locally during network outage and synced on reconnection
- Admin PIN is required to exit kiosk mode
- Kiosk works in landscape orientation on 10" tablets
- Multiple clock-ins in rapid succession (shift change) are handled without UI blocking

### Task 12.3: Offline Sync

**What:** Implement offline-first clock-in for both mobile app and kiosk mode. Clock events are stored locally and synced to the server when connectivity is restored. Conflict resolution handles edge cases.

**Design:**

```typescript
// Offline queue stored in device-local SQLite (mobile) or IndexedDB (kiosk web)
// Sync protocol:
// 1. On clock event: store locally with pending status and local timestamp
// 2. On connectivity: POST to /api/v1/time-entries/batch with all pending events
// 3. Server validates each event and returns success/conflict for each
// 4. Conflicts (e.g., duplicate clock-in) are flagged for manager review
// 5. Successfully synced events are removed from local queue
```

**Testing:**
- Clock-in succeeds in airplane mode (stored locally)
- When connectivity returns, pending events sync automatically
- Server receives events with correct local timestamps (not sync time)
- Duplicate detection: if an event was somehow submitted twice, server returns conflict
- Sync progress indicator shows X events pending
- Events sync in chronological order (oldest first)
- Failed sync retries with exponential backoff
- Local queue persists across app restart

---

## Definition of Done

A phase is considered **done** when ALL of the following criteria are met:

### Code Quality
- [ ] All code is written in TypeScript with strict mode enabled
- [ ] No `any` types except where explicitly justified with a comment
- [ ] ESLint passes with zero warnings
- [ ] All public functions and complex logic have JSDoc comments
- [ ] JSONB columns have corresponding JSON Schema validation in the application layer

### Testing
- [ ] Unit tests cover all business logic (overtime engine, state machines, validators) with >= 90% line coverage
- [ ] Integration tests cover all API endpoints with authentication and authorization checks
- [ ] E2E tests cover the critical user paths (clock-in, timesheet submit, approval)
- [ ] All tests pass in CI (GitHub Actions)
- [ ] Edge cases listed in each task's "Testing" section are covered by named test cases

### Security
- [ ] OWASP API Security Top 10 risks are addressed for all new endpoints
- [ ] Tenant isolation is verified: no cross-tenant data leakage in any query
- [ ] Authentication is required on all endpoints except /health and /oauth/token
- [ ] Rate limiting is configured on all public endpoints
- [ ] Biometric data is encrypted at rest and never transmitted to external services
- [ ] Audit events are created for all compliance-critical operations

### Documentation
- [ ] OpenAPI spec is updated for all new/modified endpoints
- [ ] JSON Schema documents are updated for any JSONB column changes
- [ ] API changelog entry describes breaking changes (if any)
- [ ] Deployment documentation is updated for any new environment variables or dependencies

### Deployment
- [ ] Docker Compose configuration builds and runs successfully
- [ ] Database migration runs without errors on an existing database
- [ ] No downtime migration: schema changes are backward-compatible
- [ ] Health check endpoint returns 200 with database connectivity confirmed

### Compliance
- [ ] FLSA overtime calculations are verified against the test cases in Task 3.1
- [ ] Audit trail is complete for all time entry modifications
- [ ] Biometric consent workflow is enforced before any biometric data collection
- [ ] GDPR erasure workflow destroys biometric data when triggered
