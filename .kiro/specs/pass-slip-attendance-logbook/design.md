# Design Document — Pass Slip & Attendance Logbook

## Overview

The Pass Slip & Attendance Logbook module is a server-rendered web application that replaces the DOST Regional Office's Excel/VBA workbook for three administrative records functions:

1. **Pass Slip (FORM7)** — dual-leg departure/return tracking with official PDF output.
2. **Attendance Logbook** — daily AM/PM time entries for permanent, contractual, and OJT employees.
3. **Guard Duty Accomplishment Report** — monthly narrative reports filed by security guards.

The system is deployed entirely on-premises on government-managed infrastructure. It has no cloud dependencies, no external integrations, and no HRIS sync. All data lives in a single PostgreSQL instance on the same local area network as the application server.

Key design constraints:
- Every write operation produces an append-only audit entry; write and audit must commit atomically.
- Soft-delete (void) with mandatory reason is the only way to remove records.
- Control numbers are server-generated, collision-free, and sequential per year-month per module.
- The guard kiosk authenticates with a shared credential but every action is attributed to a named guard selected at action time.
- PDF output is server-side only, matching the official FORM7 two-copy layout.
- The schema must accept back-dated records to 2020-01-01 to support a future data migration spec.

---

## Architecture

### Deployment Topology

```
┌─────────────────────────────────────────────────┐
│              On-Premises LAN                    │
│                                                 │
│  ┌──────────────┐      ┌─────────────────────┐  │
│  │  Guard       │      │  Admin / HR / Super  │  │
│  │  Kiosk PC    │      │  Desktop / Laptop    │  │
│  └──────┬───────┘      └──────────┬──────────┘  │
│         │   HTTP (LAN)            │              │
│         └──────────┬──────────────┘              │
│                    │                             │
│          ┌─────────▼──────────┐                 │
│          │   Application      │                 │
│          │   Server           │                 │
│          │  (Node.js / Next)  │                 │
│          │                    │                 │
│          │  ┌──────────────┐  │                 │
│          │  │ PDF Generator│  │                 │
│          │  │  (Puppeteer) │  │                 │
│          │  └──────────────┘  │                 │
│          └─────────┬──────────┘                 │
│                    │                             │
│          ┌─────────▼──────────┐                 │
│          │   PostgreSQL 15    │                 │
│          │   (same host or    │                 │
│          │    LAN host)       │                 │
│          └────────────────────┘                 │
└─────────────────────────────────────────────────┘
```

All clients are standard web browsers on the local network. There is no public internet exposure.

### Component Overview

| Component | Responsibility |
|---|---|
| **Next.js App Router** | Server-side rendered UI pages, API Route Handlers, session middleware |
| **Pass_Slip_Module** | Pass slip CRUD, control number dispatch, PDF trigger, RBAC enforcement |
| **Attendance_Module** | Attendance log CRUD, upsert logic, CSV/XLSX export |
| **Guard_Report_Module** | Guard report CRUD, period overlap check |
| **Control_Number_Service** | Atomic sequence increment per `(module, year, month)` |
| **Audit_Logger** | Append-only audit entry writer; write operations roll back if it fails |
| **PDF_Generator** | Server-side Puppeteer rendering of FORM7 HTML template |
| **Auth Layer** | Username/password login, server-side session (iron-session / next-auth credentials) |
| **RBAC Middleware** | Route-level and action-level role enforcement |
| **PostgreSQL 15** | Sole persistence layer; no external services |

---

## Technology Stack

All choices prioritise self-hostability on commodity hardware with no internet dependency.

| Layer | Choice | Rationale |
|---|---|---|
| **Runtime** | Node.js 20 LTS | Long-term support, wide ecosystem, runs on any Linux/Windows server |
| **Framework** | Next.js 14 (App Router) | Server-side rendering, API routes, no separate backend process needed |
| **Database** | PostgreSQL 15 | Mature, self-hostable, strong transactions, row-level locking for sequences |
| **ORM / Query** | Drizzle ORM | Type-safe SQL, thin abstraction, no hidden magic, easy raw SQL escape hatches |
| **Authentication** | next-auth v5 (Credentials provider) | Username/password, server-side session cookies, no external OAuth |
| **Session Store** | PostgreSQL (next-auth adapter) or iron-session (signed cookie) | No Redis dependency; session data stays on-premises |
| **PDF Generation** | Puppeteer (headless Chromium, self-hosted) | Server-side, renders pixel-accurate HTML/CSS templates, no cloud PDF API |
| **CSV/XLSX Export** | ExcelJS | Generates both CSV and XLSX server-side; no cloud dependency |
| **UI Components** | shadcn/ui + Tailwind CSS | Accessible, zero external font CDN dependencies (fonts bundled locally) |
| **Property-Based Tests** | fast-check (TypeScript) | Well-maintained PBT library, runs in Jest/Vitest, no external services |
| **Unit/Integration Tests** | Vitest + Testing Library | Fast, ESM-native, matches Next.js ecosystem |

### Dependency Notes

- **No cloud APIs**: Puppeteer bundles Chromium; ExcelJS generates files in-memory.
- **No CDN fonts**: Tailwind configured to use system fonts or bundled web fonts.
- **PostgreSQL** is the only networked dependency; everything else runs on the app server process.

---

## Components and Interfaces

### Pass_Slip_Module

Handles the full lifecycle of a pass slip record.

**Internal interface:**

```typescript
interface PassSlipService {
  create(dto: CreatePassSlipDto, actor: Actor): Promise<PassSlip>;
  submitReturnLeg(id: string, dto: ReturnLegDto, actor: Actor): Promise<PassSlip>;
  edit(id: string, dto: EditPassSlipDto, actor: Actor): Promise<PassSlip>;
  void(id: string, reason: string, actor: Actor): Promise<PassSlip>;
  search(filters: PassSlipFilters, actor: Actor): Promise<PagedResult<PassSlipSummary>>;
  exportPdf(id: string, actor: Actor): Promise<Buffer>;
}
```

### Attendance_Module

```typescript
interface AttendanceService {
  upsert(dto: AttendanceEntryDto, actor: Actor): Promise<AttendanceLog>;
  edit(id: string, dto: EditAttendanceDto, actor: Actor): Promise<AttendanceLog>;
  void(id: string, reason: string, actor: Actor): Promise<AttendanceLog>;
  search(filters: AttendanceFilters, actor: Actor): Promise<PagedResult<AttendanceLogRow>>;
  export(filters: AttendanceFilters, format: 'csv' | 'xlsx', actor: Actor): Promise<Buffer>;
}
```

### Guard_Report_Module

```typescript
interface GuardReportService {
  create(dto: CreateGuardReportDto, actor: Actor): Promise<GuardReport>;
  edit(id: string, dto: EditGuardReportDto, actor: Actor): Promise<GuardReport>;
  void(id: string, reason: string, actor: Actor): Promise<GuardReport>;
  list(guardId: string, actor: Actor): Promise<GuardReport[]>;
}
```

### Actor Type

The `Actor` type is attached to every service call and carries the resolved identity for both RBAC and audit attribution.

```typescript
interface Actor {
  sessionUserId: string;   // The logged-in user (may be kiosk account)
  sessionUserRole: Role;   // Role of the session user
  actingGuardId?: string;  // Named guard selected at action time (kiosk sessions)
  // Attribution rule: audit entry uses actingGuardId when present, else sessionUserId
}

type Role = 'employee' | 'guard' | 'supervisor' | 'admin' | 'kiosk';
```

### Control_Number_Service

```typescript
interface ControlNumberService {
  // Atomically increments and returns next sequence; throws if exhausted
  next(module: 'pass_slip', year: number, month: number): Promise<string>;
}
```

### Audit_Logger

```typescript
interface AuditLogger {
  log(entry: AuditEntry, tx: Transaction): Promise<void>;
  // Runs INSIDE the caller's transaction; failure causes caller to roll back
}

interface AuditEntry {
  tableName: string;
  recordId: string;
  operation: 'create' | 'edit' | 'void';
  beforeState: Record<string, unknown> | null;
  afterState: Record<string, unknown>;
  actingUserId: string;   // Always the acting attribution identity (never kiosk account)
  actingGuardId?: string; // Set when session is kiosk and guard was selected
  createdAt: Date;        // UTC
}
```

### PDF_Generator

```typescript
interface PdfGenerator {
  renderPassSlip(passSlip: PassSlipFull): Promise<Buffer>;
  // Returns a Buffer of the PDF bytes; throws on any Puppeteer error
}
```

---

## Data Models

### Full DDL (PostgreSQL 15)

```sql
-- ─────────────────────────────────────────────
-- Enumerated types
-- ─────────────────────────────────────────────
CREATE TYPE user_role AS ENUM (
  'employee', 'guard', 'supervisor', 'admin', 'kiosk'
);

CREATE TYPE pass_slip_type AS ENUM ('official', 'personal');

CREATE TYPE pass_slip_status AS ENUM (
  'draft', 'open', 'completed', 'voided'
);

CREATE TYPE audit_operation AS ENUM ('create', 'edit', 'void');

CREATE TYPE employee_type AS ENUM (
  'permanent', 'contractual', 'ojt'
);

CREATE TYPE approval_status AS ENUM (
  'pending', 'approved', 'rejected'
);

-- ─────────────────────────────────────────────
-- Users (includes kiosk account and named guards)
-- ─────────────────────────────────────────────
CREATE TABLE users (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username    TEXT NOT NULL UNIQUE,
  name        TEXT NOT NULL,
  password_hash TEXT NOT NULL,
  role        user_role NOT NULL,
  is_active   BOOLEAN NOT NULL DEFAULT TRUE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
-- Note: the kiosk account has role='kiosk'; its id must never appear
-- in outbound_guard_id, return_guard_id, or recorded_by.

-- ─────────────────────────────────────────────
-- Employee master list (standalone, no HRIS sync)
-- ─────────────────────────────────────────────
CREATE TABLE employees (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT NOT NULL CHECK (char_length(name) BETWEEN 1 AND 100),
  position    TEXT NOT NULL CHECK (char_length(position) BETWEEN 1 AND 100),
  division    TEXT NOT NULL CHECK (char_length(division) BETWEEN 1 AND 100),
  is_active   BOOLEAN NOT NULL DEFAULT TRUE,
  void_reason TEXT CHECK (void_reason IS NULL OR
                char_length(void_reason) BETWEEN 1 AND 500),
  voided_by   UUID REFERENCES users(id),
  voided_at   TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ─────────────────────────────────────────────
-- Control number sequences (one row per module+year+month)
-- ─────────────────────────────────────────────
CREATE TABLE control_number_sequences (
  id          BIGSERIAL PRIMARY KEY,
  module      TEXT NOT NULL DEFAULT 'pass_slip',
  year        SMALLINT NOT NULL,
  month       SMALLINT NOT NULL CHECK (month BETWEEN 1 AND 12),
  last_seq    INTEGER NOT NULL DEFAULT 0
                CHECK (last_seq BETWEEN 0 AND 9999999),
  UNIQUE (module, year, month)
);
```

```sql
-- ─────────────────────────────────────────────
-- Pass Slips
-- ─────────────────────────────────────────────
CREATE TABLE pass_slips (
  id                       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  control_no               TEXT NOT NULL UNIQUE,
  employee_id              UUID NOT NULL REFERENCES employees(id),
  position                 TEXT NOT NULL,           -- snapshot at filing time
  date                     DATE NOT NULL,           -- accepts back-dates to 2020-01-01
  time_out                 TIME NOT NULL,
  outbound_guard_id        UUID NOT NULL REFERENCES users(id),
  type                     pass_slip_type NOT NULL,
  destination              TEXT CHECK (destination IS NULL OR
                             char_length(destination) <= 500),
  purpose                  TEXT CHECK (purpose IS NULL OR
                             char_length(purpose) <= 500),
  immediate_supervisor_name TEXT CHECK (
                             immediate_supervisor_name IS NULL OR
                             char_length(immediate_supervisor_name) <= 100),
  status                   pass_slip_status NOT NULL DEFAULT 'open',
  -- Return leg (null until completed)
  time_in                  TIME,
  return_guard_id          UUID REFERENCES users(id),
  output_on_return         TEXT,
  -- Future approval gate (nullable in v1)
  approver_id              UUID REFERENCES users(id),
  approval_status          approval_status,
  -- Soft-delete
  void_reason              TEXT CHECK (void_reason IS NULL OR
                             char_length(void_reason) BETWEEN 1 AND 500),
  voided_by                UUID REFERENCES users(id),
  voided_at                TIMESTAMPTZ,
  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  -- Business rules enforced at DB level as belt-and-suspenders
  CONSTRAINT official_requires_destination_purpose
    CHECK (
      type <> 'official' OR
      (destination IS NOT NULL AND purpose IS NOT NULL)
    ),
  CONSTRAINT time_in_after_time_out
    CHECK (
      time_in IS NULL OR date IS DISTINCT FROM date OR time_in >= time_out
    )
);

CREATE INDEX idx_pass_slips_employee_id ON pass_slips(employee_id);
CREATE INDEX idx_pass_slips_date        ON pass_slips(date DESC);
CREATE INDEX idx_pass_slips_status      ON pass_slips(status);
CREATE INDEX idx_pass_slips_control_no  ON pass_slips(control_no);

-- ─────────────────────────────────────────────
-- Attendance Logs
-- ─────────────────────────────────────────────
CREATE TABLE attendance_logs (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  employee_id   UUID NOT NULL REFERENCES employees(id),
  employee_type employee_type NOT NULL,
  date          DATE NOT NULL,           -- accepts back-dates to 2020-01-01
  am_in         TIME,
  am_out        TIME,
  pm_in         TIME,
  pm_out        TIME,
  remarks       TEXT CHECK (remarks IS NULL OR char_length(remarks) <= 500),
  recorded_by   UUID NOT NULL REFERENCES users(id),
  -- Soft-delete
  void_reason   TEXT CHECK (void_reason IS NULL OR
                  char_length(void_reason) BETWEEN 1 AND 500),
  voided_by     UUID REFERENCES users(id),
  voided_at     TIMESTAMPTZ,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  -- Either all four time fields present or all absent
  CONSTRAINT all_times_or_none CHECK (
    (am_in IS NOT NULL AND am_out IS NOT NULL AND
     pm_in IS NOT NULL AND pm_out IS NOT NULL)
    OR
    (am_in IS NULL AND am_out IS NULL AND
     pm_in IS NULL AND pm_out IS NULL)
  ),
  -- Upsert uniqueness: one record per employee per date (non-voided)
  UNIQUE (employee_id, date)
);

CREATE INDEX idx_attendance_employee_date
  ON attendance_logs(employee_id, date DESC);
CREATE INDEX idx_attendance_date
  ON attendance_logs(date DESC);

-- ─────────────────────────────────────────────
-- Guard Duty Accomplishment Reports
-- ─────────────────────────────────────────────
CREATE TABLE guard_reports (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  guard_id     UUID NOT NULL REFERENCES users(id),
  period_start DATE NOT NULL,
  period_end   DATE NOT NULL,
  narrative    TEXT NOT NULL CHECK (char_length(narrative) BETWEEN 1 AND 10000),
  -- Soft-delete
  void_reason  TEXT CHECK (void_reason IS NULL OR
                 char_length(void_reason) BETWEEN 1 AND 500),
  voided_by    UUID REFERENCES users(id),
  voided_at    TIMESTAMPTZ,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CONSTRAINT period_end_not_before_start
    CHECK (period_end >= period_start)
);

CREATE INDEX idx_guard_reports_guard_id
  ON guard_reports(guard_id);
CREATE INDEX idx_guard_reports_period
  ON guard_reports(period_start, period_end);
```

```sql
-- ─────────────────────────────────────────────
-- Audit Log (append-only)
-- ─────────────────────────────────────────────
CREATE TABLE audit_log (
  id             BIGSERIAL PRIMARY KEY,
  table_name     TEXT NOT NULL,
  record_id      UUID NOT NULL,
  operation      audit_operation NOT NULL,
  before_state   JSONB,          -- null for 'create' operations
  after_state    JSONB NOT NULL,
  acting_user_id UUID NOT NULL REFERENCES users(id),
  acting_guard_id UUID REFERENCES users(id),
  -- acting_guard_id is set when session is kiosk and a named guard was selected;
  -- for kiosk sessions this is the true attribution identity.
  created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
  -- No UPDATE or DELETE privileges granted on this table.
);

CREATE INDEX idx_audit_log_record
  ON audit_log(table_name, record_id, created_at ASC);
CREATE INDEX idx_audit_log_acting_user
  ON audit_log(acting_user_id, created_at DESC);
```

### Entity Relationship Summary

```
users ──< pass_slips (outbound_guard_id, return_guard_id, approver_id, voided_by)
employees ──< pass_slips (employee_id)
employees ──< attendance_logs (employee_id)
users ──< attendance_logs (recorded_by, voided_by)
users ──< guard_reports (guard_id, voided_by)
users ──< audit_log (acting_user_id, acting_guard_id)
control_number_sequences (module + year + month → last_seq)
```

### Attribution Rule (Kiosk Sessions)

When the session belongs to the `kiosk` account:

| Field | Value stored |
|---|---|
| `outbound_guard_id` | Selected named guard's `users.id` |
| `return_guard_id` | Selected named guard's `users.id` |
| `recorded_by` (attendance) | Selected named guard's `users.id` |
| `audit_log.acting_user_id` | Selected named guard's `users.id` |
| `audit_log.acting_guard_id` | Selected named guard's `users.id` |

The kiosk account's `users.id` is **never** stored in any of these fields.

---

## API Design

All routes are Next.js App Router Route Handlers under `/app/api/`. Authentication is verified by middleware before each handler runs. RBAC is checked at the handler level.

### Authentication

| Method | Route | Description |
|---|---|---|
| `POST` | `/api/auth/login` | Username/password login; sets server-side session cookie |
| `POST` | `/api/auth/logout` | Destroys session |
| `GET` | `/api/auth/me` | Returns current session user and role |

### Guard Selection (Kiosk)

| Method | Route | Description |
|---|---|---|
| `GET` | `/api/guards` | Lists active users with role `guard`; used to populate guard picker |

### Pass Slips

| Method | Route | Description | Roles |
|---|---|---|---|
| `POST` | `/api/pass-slips` | Create outbound leg | employee, guard, admin |
| `GET` | `/api/pass-slips` | Search/filter (query params below) | guard, supervisor, admin; employee sees own only |
| `GET` | `/api/pass-slips/:id` | Fetch single record | guard, supervisor, admin; employee sees own only |
| `PATCH` | `/api/pass-slips/:id/return` | Submit return leg | guard, admin |
| `PATCH` | `/api/pass-slips/:id` | Edit fields | admin |
| `POST` | `/api/pass-slips/:id/void` | Void with reason | admin |
| `GET` | `/api/pass-slips/:id/pdf` | Download PDF | guard, supervisor, admin |

**Search query parameters** (`GET /api/pass-slips`):

```
date_from      YYYY-MM-DD  (inclusive)
date_to        YYYY-MM-DD  (inclusive)
employee_name  string      (partial match, min 2 chars)
status         open | completed | voided | (omit = exclude voided)
page           integer     (default 1)
page_size      integer     (default 20, max 100)
```

### Attendance Logs

| Method | Route | Description | Roles |
|---|---|---|---|
| `POST` | `/api/attendance` | Create or update entry (upsert) | guard, admin |
| `GET` | `/api/attendance` | Filter/search | admin, supervisor, guard |
| `PATCH` | `/api/attendance/:id` | Edit entry | admin; guard for current date only |
| `POST` | `/api/attendance/:id/void` | Void with reason | admin |
| `GET` | `/api/attendance/export` | Download CSV/XLSX (same filters) | admin, supervisor |

### Guard Reports

| Method | Route | Description | Roles |
|---|---|---|---|
| `POST` | `/api/guard-reports` | File new report | guard, admin |
| `GET` | `/api/guard-reports` | List own reports | guard; admin sees all |
| `GET` | `/api/guard-reports/:id` | Fetch single report | guard (own), admin |
| `PATCH` | `/api/guard-reports/:id` | Edit narrative/period | guard (own), admin |
| `POST` | `/api/guard-reports/:id/void` | Void with reason | admin |

### Employees (Admin)

| Method | Route | Description | Roles |
|---|---|---|---|
| `POST` | `/api/employees` | Create employee | admin |
| `GET` | `/api/employees` | List/search employees | all authenticated |
| `PATCH` | `/api/employees/:id` | Edit employee | admin |
| `POST` | `/api/employees/:id/void` | Soft-delete | admin |

### Audit Trail

| Method | Route | Description | Roles |
|---|---|---|---|
| `GET` | `/api/audit/:table/:recordId` | Full change history for a record | admin |

### Users (Admin)

| Method | Route | Description | Roles |
|---|---|---|---|
| `POST` | `/api/users` | Create user | admin |
| `GET` | `/api/users` | List users | admin |
| `PATCH` | `/api/users/:id` | Edit user (name, role, active) | admin |

### Standard Response Envelope

```json
{
  "success": true,
  "data": { ... },
  "meta": { "page": 1, "pageSize": 20, "total": 143 }
}
```

Error response:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable summary",
    "fields": { "field_name": "specific reason" }
  }
}
```

---

## Control Number Generation Algorithm

### Format

```
YYYY-MM-#######
```

- `YYYY` = 4-digit year from the Pass Slip's `date` field (not the current wall-clock date).
- `MM` = zero-padded 2-digit month from `date`.
- `#######` = 7-digit zero-padded sequential counter, starting at `0000001`, scoped to `(module, year, month)`.
- Maximum value: `9999999`. Any attempt to exceed this returns an error.
- Gaps are allowed (failed transactions do not roll back the sequence).

### Concurrency-Safe Implementation

The sequence increment runs inside the same PostgreSQL transaction that creates the pass slip. A `SELECT ... FOR UPDATE` advisory lock on the sequence row (or `UPDATE ... RETURNING`) ensures no two concurrent transactions receive the same counter value.

```sql
-- Step 1: Insert or advance sequence (upsert)
INSERT INTO control_number_sequences (module, year, month, last_seq)
VALUES ($module, $year, $month, 1)
ON CONFLICT (module, year, month)
DO UPDATE
  SET last_seq = control_number_sequences.last_seq + 1
  WHERE control_number_sequences.last_seq < 9999999
RETURNING last_seq;

-- If the UPDATE matched zero rows (last_seq already at 9999999),
-- the RETURNING clause returns no row → application raises EXHAUSTED error.
```

The `RETURNING last_seq` from this single statement is the only source of truth for the new sequence number. No separate SELECT is needed. Because PostgreSQL processes this as an atomic operation under row-level locking, two simultaneous transactions for the same `(module, year, month)` are serialised at the database level — one blocks until the other commits or rolls back.

**Algorithm in application code:**

```typescript
async function nextControlNumber(
  tx: Transaction,
  module: string,
  year: number,
  month: number
): Promise<string> {
  const result = await tx.execute(sql`
    INSERT INTO control_number_sequences (module, year, month, last_seq)
    VALUES (${module}, ${year}, ${month}, 1)
    ON CONFLICT (module, year, month)
    DO UPDATE
      SET last_seq = control_number_sequences.last_seq + 1
      WHERE control_number_sequences.last_seq < 9999999
    RETURNING last_seq
  `);

  if (result.rows.length === 0) {
    throw new ControlNumberExhaustedError(year, month);
  }

  const seq: number = result.rows[0].last_seq;
  const mm = String(month).padStart(2, '0');
  const seqStr = String(seq).padStart(7, '0');
  return `${year}-${mm}-${seqStr}`;
}
```

This function is always called **inside** the same database transaction as the `INSERT INTO pass_slips`. If the pass slip insert fails and the transaction rolls back, the `control_number_sequences.last_seq` roll back also — but because gaps are allowed by requirement, this is acceptable. A failed transaction permanently consumes the sequence slot.

---

## PDF Generation

### Approach

PDF output is produced server-side by Puppeteer (headless Chromium, bundled with the application). An HTML template is rendered with the pass slip data, then Puppeteer converts the page to PDF.

No client-side PDF generation is used. The PDF bytes are streamed directly from the API response to the browser for download.

### FORM7 Layout

The template (`/src/templates/form7.html`) produces a single A4 page containing two stacked identical copies:

```
┌─────────────────────────────────────┐
│         DOST Regional Office        │
│         Pass Slip (FORM7)           │
│         ORD's Copy                  │
│─────────────────────────────────────│
│  Control No:  YYYY-MM-#######       │
│  Name:        [employee_name]       │
│  Position:    [position]            │
│  Date:        [date]                │
│  Time Out:    [time_out]            │
│  Time In:     [time_in]             │
│  Type:        [Official/Personal]   │
│  Destination: [destination]         │
│  Purpose:     [purpose]             │
│  Output:      [output_on_return]    │
│─────────────────────────────────────│
│  Guard on Duty: ________________    │
│  Immediate Supervisor: __________   │
│  Employee Signature: ____________   │
├─────────────────────────────────────┤  ← cut line
│         DOST Regional Office        │
│         Pass Slip (FORM7)           │
│         Guard's Copy                │
│  [same fields as above]             │
└─────────────────────────────────────┘
```

### Watermarks

| Status | Watermark text | Behaviour |
|---|---|---|
| `voided` | `VOID` | Diagonal red, opacity 0.35, rendered on both copies |
| `draft` | `DRAFT` | Diagonal grey, opacity 0.25, rendered on both copies |
| `open` / `completed` | None | No watermark |

The watermark is a CSS `position: absolute` element with `transform: rotate(-45deg)`, large font size, and `pointer-events: none`, placed over each copy's container.

### Error Handling

If Puppeteer throws during generation, the API route catches the error, logs it server-side, and returns HTTP 500 with a JSON error body. No partial PDF bytes are sent to the client.

---

## Audit Trail Implementation

### Invariant

Every write operation (create, edit, void) **must** produce an audit entry within the same database transaction. If the audit entry cannot be persisted, the write is rolled back. This is enforced by passing the open transaction to `AuditLogger.log()` — if `AuditLogger.log()` throws, the transaction is rolled back by the caller's `catch` block before returning an error to the client.

### Audit Entry Content

| Field | Content |
|---|---|
| `table_name` | `'pass_slips'` / `'attendance_logs'` / `'guard_reports'` / `'employees'` |
| `record_id` | UUID of the affected row |
| `operation` | `'create'` / `'edit'` / `'void'` |
| `before_state` | Full JSONB row snapshot before change; `null` for creates |
| `after_state` | Full JSONB row snapshot after change |
| `acting_user_id` | Named guard's ID (kiosk sessions) or session user's ID (others) |
| `acting_guard_id` | Named guard's ID when session is kiosk; otherwise null |
| `created_at` | `NOW()` (UTC, from PostgreSQL clock) |

### Transaction Pattern (TypeScript pseudocode)

```typescript
async function editPassSlip(id: string, dto: EditDto, actor: Actor) {
  await db.transaction(async (tx) => {
    const before = await tx.query.passSlips.findFirst({ where: eq(passSlips.id, id) });
    if (!before) throw new NotFoundError();
    if (before.status === 'voided') throw new ValidationError('Voided records cannot be modified');

    const after = await tx.update(passSlips)
      .set({ ...dto, updatedAt: new Date() })
      .where(eq(passSlips.id, id))
      .returning();

    // AuditLogger runs inside the SAME transaction.
    // If this throws, the entire transaction rolls back.
    await auditLogger.log({
      tableName: 'pass_slips',
      recordId: id,
      operation: 'edit',
      beforeState: before,
      afterState: after[0],
      actingUserId: actor.actingGuardId ?? actor.sessionUserId,
      actingGuardId: actor.actingGuardId,
      createdAt: new Date(),
    }, tx);
  });
}
```

### Read-Only Guarantee

The `audit_log` table is created with no `UPDATE` or `DELETE` permissions for the application database user. The application user only has `INSERT` and `SELECT` on `audit_log`. This is enforced at the PostgreSQL role level:

```sql
-- Application role setup
GRANT SELECT, INSERT ON audit_log TO app_user;
-- No UPDATE, DELETE, TRUNCATE granted
```

---

## Kiosk Authentication Flow

The guard station operates a shared browser session authenticated with the kiosk credential. Every sensitive action requires the operator to additionally supply the named guard's identity.

### Session Login

```
1. Operator opens browser to /login
2. Enters kiosk username + password
3. Server verifies credentials, issues HttpOnly session cookie
4. Session record: { userId: <kiosk-uuid>, role: 'kiosk' }
5. Browser is redirected to /kiosk/dashboard
```

### Guard Selection Before Each Action

```
1. Operator selects action (e.g., "Record Pass Slip")
2. System displays guard picker: dropdown of active users with role='guard'
3. Operator selects their name (e.g., "SGT. Juan dela Cruz")
4. Selected guardId is included in the request body as guard_id
5. Server-side validation:
   a. Session must be valid (any role)
   b. guard_id must be present and reference an active role='guard' user
   c. The session userId (kiosk account) is NOT used in any attribution field
6. Action proceeds with actor = { sessionUserId: <kiosk-uuid>, sessionUserRole: 'kiosk',
                                   actingGuardId: <guard-uuid> }
```

### Attribution Enforcement (Server-Side Middleware)

```typescript
function resolveActor(session: Session, body: RequestBody): Actor {
  if (session.role === 'kiosk') {
    const guardId = body.guard_id;
    if (!guardId) throw new ValidationError('guard_id is required for kiosk sessions');
    // Validated against DB inside service layer
    return {
      sessionUserId: session.userId,
      sessionUserRole: 'kiosk',
      actingGuardId: guardId,
    };
  }
  return {
    sessionUserId: session.userId,
    sessionUserRole: session.role,
    actingGuardId: undefined,
  };
}
```

The rule that "kiosk account ID never appears in attribution fields" is enforced in the service layer by always using `actor.actingGuardId ?? actor.sessionUserId` for attribution fields, and separately validating that `actingGuardId` is present whenever the session role is `kiosk`.

---

## Role-Based Access Control Implementation

### Role Matrix

| Action | employee | guard | supervisor | admin | kiosk\* |
|---|---|---|---|---|---|
| File Pass Slip outbound | ✓ (own) | ✓ | — | ✓ | ✓ (with guard_id) |
| File Pass Slip return leg | — | ✓ | — | ✓ | ✓ (with guard_id) |
| Edit Pass Slip | — (output_on_return only) | — | — | ✓ | — |
| Void Pass Slip | — | — | — | ✓ | — |
| View Pass Slips | own only | ✓ | ✓ | ✓ | ✓ |
| Export Pass Slip PDF | — | ✓ | ✓ | ✓ | ✓ |
| File Attendance entry | — | ✓ | — | ✓ | ✓ (with guard_id) |
| Edit Attendance (today) | — | ✓ | — | ✓ | ✓ (with guard_id) |
| Edit Attendance (past) | — | — | — | ✓ | — |
| Void Attendance | — | — | — | ✓ | — |
| File Guard Report | — | ✓ (own) | — | ✓ | — |
| Edit Guard Report | — | ✓ (own) | — | ✓ | — |
| Void Guard Report | — | — | — | ✓ | — |
| Manage Employee Master | — | — | — | ✓ | — |
| Manage Users | — | — | — | ✓ | — |
| View Audit Trail | — | — | — | ✓ | — |

\* kiosk role inherits guard-level permissions but always requires `guard_id` in request body.

### RBAC Middleware Structure

```typescript
// /src/middleware/rbac.ts
export function requireRole(...roles: Role[]) {
  return async (req: Request, actor: Actor, next: () => Promise<Response>) => {
    const effectiveRole = actor.sessionUserRole === 'kiosk'
      ? 'guard'   // kiosk is treated as guard for RBAC purposes
      : actor.sessionUserRole;
    if (!roles.includes(effectiveRole)) {
      return jsonError(403, 'INSUFFICIENT_PERMISSIONS',
        `Role ${actor.sessionUserRole} is not permitted to perform this action`);
    }
    return next();
  };
}
```

The middleware maps `kiosk` → `guard` for RBAC evaluation. Any action requiring `guard` access is also accessible from a kiosk session, provided `guard_id` is supplied.

Employee self-service scoping: Pass Slip search for `employee` role automatically appends `AND employee_id = <session.userId's linked employee record>` to all queries.

---

## Key UI Flows

### 1. Pass Slip Filing (Outbound Leg)

```
User: employee, guard, or kiosk (guard selects own name first)

1. Navigate to /pass-slips/new
2. [Kiosk only] Guard picker appears → operator selects named guard
3. Form displayed:
   - Employee selector (searchable by name)
   - Date (defaults to today)
   - Time Out (defaults to current time HH:MM)
   - Type: Official / Personal (radio)
   - [If Official] Destination (required), Purpose (required)
   - Immediate Supervisor Name (free text, optional)
4. Submit → POST /api/pass-slips
5. Server: validate → generate control_no → insert pass_slip → insert audit_log (same tx)
6. On success: redirect to /pass-slips/<id> with banner showing control number
7. On validation error: form re-displays with field-level error messages
```

### 2. Pass Slip Return Leg

```
User: guard or admin (kiosk with guard selection)

1. Navigate to /pass-slips (search by employee name or date)
2. Locate record with status "open"
3. Click "Record Return"
4. [Kiosk only] Guard picker confirmation
5. Form displayed:
   - Time In (HH:MM, validated > time_out)
   - Return Guard (auto-populated from selected guard; editable by admin)
   - Output on Return (optional free text)
6. Submit → PATCH /api/pass-slips/:id/return
7. Server: validate time_in > time_out → update status to 'completed' → audit log
8. On success: pass slip detail page updated, "Print PDF" button becomes available
```

### 3. Attendance Entry

```
User: guard or admin (kiosk with guard selection)

1. Navigate to /attendance/new (or /attendance/date/:YYYY-MM-DD)
2. [Kiosk only] Guard picker
3. Date selector (defaults to today)
4. Employee list for selected date loads (searchable)
5. For each employee row:
   - Employee Type selector (permanent / contractual / ojt)
   - am_in, am_out, pm_in, pm_out (time inputs, all-or-none per row)
   - Remarks
6. Save row → POST /api/attendance (upsert)
7. If record exists for employee+date: UPDATE with audit; else: INSERT with audit
8. Validation errors per row are highlighted inline
```

### 4. Guard Report Filing

```
User: guard (own reports only) or admin

1. Navigate to /guard-reports/new
2. Form displayed:
   - Guard (pre-populated from session; admin can change)
   - Period Start / Period End (date pickers)
   - Narrative (textarea, 10,000 char max with character counter)
3. Submit → POST /api/guard-reports
4. Server: validate no period overlap for same guard → insert → audit log
5. On conflict: error shown with link to conflicting report's ID
```

### 5. Void Flow (Admin, all record types)

```
1. Admin locates record in list or detail view
2. Clicks "Void Record" button
3. Modal appears: "Enter void reason (required, max 500 characters)"
4. Admin enters reason → confirms
5. POST /api/[module]/:id/void with { void_reason }
6. Server: validate reason length → soft-delete → audit log
7. Record displays with "VOIDED" badge; void_reason visible in detail view
```

### 6. Audit Trail View

```
1. Admin navigates to record detail page
2. "View History" tab / section
3. GET /api/audit/:table/:recordId
4. Timeline displayed: newest first → oldest last (ordered by timestamp asc in server, reversed for display)
5. Each entry shows: timestamp, acting user name, operation, changed fields (before/after diff)
```

---

## Error Handling

### Error Taxonomy

| Code | HTTP Status | Description |
|---|---|---|
| `UNAUTHENTICATED` | 401 | No valid session |
| `INSUFFICIENT_PERMISSIONS` | 403 | Valid session, wrong role |
| `NOT_FOUND` | 404 | Record does not exist |
| `VALIDATION_ERROR` | 422 | Field-level input errors |
| `CONFLICT` | 409 | Duplicate control number space / guard report period overlap |
| `CONTROL_NUMBER_EXHAUSTED` | 422 | Year-month sequence at 9,999,999 |
| `ALREADY_VOIDED` | 409 | Attempt to void or edit an already-voided record |
| `AUDIT_FAILURE` | 500 | Audit log write failed; write was rolled back |
| `PDF_GENERATION_ERROR` | 500 | Puppeteer encountered an error |
| `INTERNAL_ERROR` | 500 | Unexpected server-side failure |

### Audit Failure Propagation

If the audit log insert fails inside a transaction, the transaction rolls back:

```typescript
try {
  await db.transaction(async (tx) => {
    await tx.insert(passSlips).values(newRecord);
    await auditLogger.log({ ... }, tx); // throws → rolls back
  });
} catch (err) {
  if (err instanceof AuditFailureError) {
    return jsonError(500, 'AUDIT_FAILURE',
      'Your changes could not be saved because the audit log could not be written.');
  }
  throw err;
}
```

The user receives a clear message that their changes were not saved and no partial record was created.

### Validation Strategy

All input is validated at the API Route Handler level before reaching the service layer:

1. **Schema validation**: Zod schemas define the shape and constraints for every DTO.
2. **Business rule validation**: Service layer checks cross-field rules (e.g., `time_in > time_out`, period overlap).
3. **Database constraints**: `CHECK` constraints in PostgreSQL act as a final safety net.

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

The feature involves significant pure logic: control number generation, validation rules, status state machines, search ordering, void semantics, and audit attribution. Property-based testing is appropriate and is implemented using **fast-check** (TypeScript).

Each property test is configured to run a minimum of **100 iterations**. Tests are tagged with: `Feature: pass-slip-attendance-logbook, Property N: <property text>`.

---

### Reflection — Pre-Consolidation Notes

Before writing final properties, redundant and overlapping candidates are resolved:

- Properties 2.1, 3.1 (create/return leg round-trips): Combined into a single "field persistence round-trip" property that covers any write operation on a pass slip.
- Properties 4.2 and 13.6 (audit atomicity): Identical invariant across modules → single "write-audit atomicity" property applicable to all three modules.
- Properties 5.5 and 5.6 (void filtering): Two sides of the same filtering invariant → one "void exclusion in default search" property.
- Properties 6.6 and 10.6 (invalid date range): Identical rule in two modules → single "invalid date range rejection" property covering both modules.
- Properties 2.2 / 2.3 / 2.4 (validation rules): Three related validation rules combined into two properties: official-type validation and all-or-none time fields.
- Properties 7.1 / 7.4 / 7.5 (PDF template): Template rendering checks combined into one "template renders required fields and watermarks" property.
- Properties 3.1 and 3.2 (return leg + status transition): Fully overlapping → one property.
- Properties 10.2 (attendance ordering) and 6.2 (pass slip ordering): Both are "result ordering" properties — kept separate because ordering fields differ (date DESC vs date ASC), but noted as the same pattern.

After reflection: **14 distinct properties** remain, each providing unique validation value.

---

### Property 1: Control Number Format Correctness

*For any* valid combination of `(year, month, sequenceNumber)` where year ≥ 2020, month ∈ [1..12], and sequenceNumber ∈ [1..9999999], the formatted control number produced by `formatControlNumber(year, month, seq)` SHALL match the regex `/^\d{4}-\d{2}-\d{7}$/`, the year segment SHALL equal the input year, the month segment SHALL be the zero-padded input month, and the sequence segment SHALL be the zero-padded 7-digit input sequence number.

**Validates: Requirements 1.1**

---

### Property 2: Control Number Uniqueness (Monotonic Sequence)

*For any* N ∈ [1..100] sequential calls to `nextControlNumber` on the same `(module, year, month)` starting from any initial `last_seq` value, all N returned control numbers SHALL be distinct strings, and each subsequent control number SHALL have a strictly greater sequence component than its predecessor.

**Validates: Requirements 1.2, 1.4**

---

### Property 3: Control Number Client Supply Rejection

*For any* pass slip creation request that includes a `control_no` field — regardless of whether the value is a valid format string, an empty string, a whitespace string, or any other non-null value — the Pass_Slip_Module SHALL reject the request with a validation error and SHALL NOT create the record.

**Validates: Requirements 1.3, 2.5**

---

### Property 4: Pass Slip Creation Round-Trip

*For any* valid `CreatePassSlipDto` (any employee_id, date from 2020-01-01 to today, time_out in [00:00..23:59], type, valid outbound_guard_id, and for official type: non-empty destination and purpose), submitting the pass slip SHALL result in a stored record where every supplied field value is retrievable by ID with an identical value, the status is `'open'`, the `approver_id` and `approval_status` are null, and the `control_no` matches the YYYY-MM-####### format.

**Validates: Requirements 2.1, 2.7, 1.1**

---

### Property 5: Official Pass Slip Requires Destination and Purpose

*For any* pass slip creation request with `type = 'official'` where `destination` is null, empty, or absent, or `purpose` is null, empty, or absent, the Pass_Slip_Module SHALL reject the submission with a validation error identifying the missing field. *For any* pass slip creation request with `type = 'personal'`, the stored record SHALL have `destination = null` and `purpose = null` regardless of any values supplied for those fields.

**Validates: Requirements 2.2, 2.3**

---

### Property 6: Return Leg Time Ordering Invariant

*For any* return leg submission where `time_in` (as a clock time on the same date) is strictly earlier than the `time_out` stored on the matching pass slip, the Pass_Slip_Module SHALL reject the submission with a validation error. *For any* return leg submission where `time_in >= time_out`, the submission SHALL be accepted, `time_in` and `return_guard_id` SHALL be persisted, and the pass slip status SHALL transition to `'completed'`.

**Validates: Requirements 3.1, 3.2, 3.4**

---

### Property 7: Return Leg Rejected for Non-Open Records

*For any* pass slip in a status other than `'open'` (i.e., `'draft'`, `'completed'`, or `'voided'`), any return leg submission targeting that pass slip SHALL be rejected with an error indicating the record is not awaiting a return entry.

**Validates: Requirements 3.3**

---

### Property 8: Write-Audit Atomicity Across All Modules

*For any* create, edit, or void operation across Pass Slip, Attendance Log, or Guard Report records, if the operation completes successfully then the `audit_log` table SHALL contain exactly one new entry for that operation with `before_state` equal to the record's full state before the change (null for creates), `after_state` equal to the record's full state after the change, `acting_user_id` equal to the attribution identity (named guard for kiosk sessions, session user otherwise), and a UTC `created_at` timestamp. *For any* operation where the audit log write fails (simulated), the target record SHALL remain unchanged and the operation SHALL return an error to the caller.

**Validates: Requirements 4.2, 9.2, 12.2, 13.1, 13.2, 13.3, 13.6**

---

### Property 9: Void Semantics — Status, Reason Persistence, and Filtering

*For any* valid void request with a `void_reason` string of length ∈ [1..500] characters, the voided record SHALL transition to status `'voided'`, the `void_reason` SHALL be stored exactly as supplied, and the record SHALL be excluded from all search results that do not explicitly filter for `status = 'voided'`. *For any* search that explicitly includes `status = 'voided'`, all voided records matching other filter criteria SHALL appear in the results. *For any* void request with an absent, empty, or length > 500 `void_reason`, the module SHALL reject and leave the record unchanged.

**Validates: Requirements 5.1, 5.2, 5.5, 5.6, 9.3, 9.4, 12.5, 12.6, 16.2, 16.3**

---

### Property 10: Search Result Ordering

*For any* result set returned by the Pass_Slip_Module search interface, records SHALL be ordered by `date` descending and, within the same date, by `control_no` descending. *For any* result set returned by the Attendance_Module filter interface, records SHALL be ordered by `date` ascending and, within the same date, by employee name ascending.

**Validates: Requirements 6.2, 6.3, 10.2**

---

### Property 11: Invalid Date Range Rejection

*For any* search or filter query in the Pass_Slip_Module or the Attendance_Module where `date_from` is strictly later than `date_to`, the module SHALL reject the query with a validation error indicating the date range is invalid, and SHALL perform no database search.

**Validates: Requirements 6.6, 10.6**

---

### Property 12: PDF Template Renders All Required Fields and Correct Watermarks

*For any* valid `PassSlip` object, the HTML template rendered by `renderPassSlipTemplate(passSlip)` SHALL contain the string values for all required FORM7 fields (control_no, employee name, position, date, time_out, type, outbound_guard_id display name, immediate_supervisor_name), the labels "ORD's Copy" and "Guard's Copy", and — when `status = 'voided'` — the watermark text "VOID"; when `status = 'draft'` — the watermark text "DRAFT"; when `status` is `'open'` or `'completed'` — no watermark text.

**Validates: Requirements 7.1, 7.2, 7.3, 7.4, 7.5**

---

### Property 13: Attendance Log Upsert Uniqueness

*For any* sequence of N ≥ 1 attendance log upsert calls with the same `(employee_id, date)` pair, the total count of non-voided `attendance_logs` rows for that `(employee_id, date)` SHALL be exactly 1 after all N calls complete, and the stored row SHALL reflect the values from the most recent successful upsert.

**Validates: Requirements 8.2**

---

### Property 14: Attendance All-or-None Time Fields

*For any* attendance log submission where exactly 1, 2, or 3 of the four time fields (`am_in`, `am_out`, `pm_in`, `pm_out`) are non-null, the Attendance_Module SHALL reject the submission with a validation error. *For any* submission where all four are provided or all four are absent/null, the submission SHALL be accepted as valid (subject to other validation passing).

**Validates: Requirements 8.6**

---

### Property 15: Guard Report Period Non-Overlap

*For any* two guard report submissions for the same `guard_id` where the periods `[period_start_1, period_end_1]` and `[period_start_2, period_end_2]` overlap (i.e., `period_start_2 <= period_end_1 AND period_start_1 <= period_end_2`), the second submission SHALL be rejected with a conflict error that includes the existing report's ID. *For any* two guard report submissions with non-overlapping periods for the same guard, both SHALL be accepted.

**Validates: Requirements 11.4, 11.5**

---

### Property 16: Kiosk Attribution Invariant

*For any* action submitted through a kiosk session with a valid `guard_id`, the stored `outbound_guard_id`, `return_guard_id`, or `recorded_by` field (depending on action type) SHALL equal the supplied `guard_id` (the named guard's user ID), and the `audit_log` entry's `acting_user_id` SHALL also equal the named guard's user ID. The kiosk account's user ID SHALL NOT appear in any of these fields in any record or audit entry.

**Validates: Requirements 18.2, 18.5, 18.6**

---

## Testing Strategy

### Dual Testing Approach

Both **unit/example-based tests** and **property-based tests** are used. Unit tests cover specific concrete scenarios and integration wiring; property tests verify the universal invariants enumerated above across hundreds of random inputs.

### Property-Based Testing Setup

**Library:** `fast-check` (TypeScript)
**Test runner:** Vitest
**Minimum iterations per property:** 100 (configured via `fc.assert(fc.property(...), { numRuns: 100 })`)
**Tag format in test comments:** `Feature: pass-slip-attendance-logbook, Property N: <property text>`

```typescript
// Example property test structure
import fc from 'fast-check';
import { describe, it } from 'vitest';

describe('Feature: pass-slip-attendance-logbook', () => {
  it('Property 1: Control Number Format Correctness', () => {
    // Feature: pass-slip-attendance-logbook, Property 1: For any valid (year, month, seq), formatted control number matches YYYY-MM-####### and encodes inputs correctly
    fc.assert(fc.property(
      fc.integer({ min: 2020, max: 2099 }),
      fc.integer({ min: 1, max: 12 }),
      fc.integer({ min: 1, max: 9_999_999 }),
      (year, month, seq) => {
        const result = formatControlNumber(year, month, seq);
        expect(result).toMatch(/^\d{4}-\d{2}-\d{7}$/);
        expect(result.startsWith(String(year))).toBe(true);
        expect(result.slice(5, 7)).toBe(String(month).padStart(2, '0'));
        expect(result.slice(8)).toBe(String(seq).padStart(7, '0'));
      }
    ), { numRuns: 100 });
  });
});
```

### Unit Test Focus Areas

Unit tests (non-property) cover:

- Specific HTTP status codes for each error type (401, 403, 404, 409, 422, 500)
- Control number exhaustion boundary (`last_seq = 9999999`)
- Employee self-service scoping (employee role sees only own records)
- Guard report period overlap with exactly-adjacent periods (edge: `[1,10]` and `[11,20]` do NOT overlap; `[1,10]` and `[10,20]` DO overlap by shared endpoint)
- `output_on_return` is optional on return leg (null is valid)
- `approval_status` and `approver_id` are null on create
- Upsert: second write to same `(employee_id, date)` updates, does not insert
- Admin may edit `output_on_return` on a completed pass slip; guard/employee may not edit other fields
- Guard cannot edit attendance for a past date
- Kiosk session without `guard_id` in body is rejected with `VALIDATION_ERROR`

### Integration Tests

Integration tests run against a real PostgreSQL instance (in a Docker container in CI, or a local test database on-premises):

- Full pass slip lifecycle: create → return → search → PDF → void
- Attendance upsert idempotency over multiple writes
- Guard report overlap detection with real date arithmetic
- Audit log entries created atomically with writes (verify rollback on mock audit failure)
- Control number sequence under simulated concurrent requests (2 concurrent inserts produce distinct values)
- Session cookie lifecycle: login → access protected route → logout → 401

### Smoke Tests

- Application database user has no `DELETE` or `UPDATE` privilege on `audit_log`
- Application database user has no `DELETE` privilege on any core table
- Puppeteer can launch headless Chromium (server startup health check)
- Login page accessible at `/login`
- PDF generation returns non-empty bytes for a valid pass slip

### Accessibility

- All form inputs have associated `<label>` elements or `aria-label` attributes
- Error messages are associated with their fields via `aria-describedby`
- Guard picker dropdown is keyboard-navigable
- Focus management: after successful form submission, focus moves to the confirmation banner or the newly created record

---

*Design document complete. Ready for task generation.*
