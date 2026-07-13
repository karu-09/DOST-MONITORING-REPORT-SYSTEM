# Implementation Plan: Pass Slip & Attendance Logbook

## Overview

Implement the Pass Slip & Attendance Logbook module as a Next.js 14 (App Router) application with PostgreSQL 15, Drizzle ORM, next-auth v5, Puppeteer PDF generation, and ExcelJS export. The build follows the layered architecture in the design: database schema → shared services → module services → API route handlers → UI pages.

## Tasks

- [x] 1. Project scaffold and database schema
  - [x] 1.1 Initialise Next.js 14 App Router project with TypeScript, Tailwind CSS, and shadcn/ui
    - Run `create-next-app` with TypeScript and App Router; install and configure shadcn/ui with Tailwind; configure Tailwind to use local/system fonts with no CDN dependency; add `@vercel/postgres` or `pg` driver plus Drizzle ORM and `drizzle-kit`; add `fast-check` and `vitest` + `@testing-library/react`; add `next-auth@5`, `puppeteer`, and `exceljs` at pinned versions
    - _Requirements: all_

  - [x] 1.2 Create Drizzle schema file with all enums, tables, constraints, and indexes
    - Translate the full DDL from the design into `src/db/schema.ts`: enums (`user_role`, `pass_slip_type`, `pass_slip_status`, `audit_operation`, `employee_type`, `approval_status`); tables `users`, `employees`, `control_number_sequences`, `pass_slips`, `attendance_logs`, `guard_reports`, `audit_log`; all `CHECK` constraints, `UNIQUE` constraints, and indexes exactly as specified in the DDL section of the design
    - _Requirements: 1.1, 2.1, 3.1, 8.1, 11.1, 13.2, 15.1, 16.1_

  - [x] 1.3 Write and run Drizzle migration to apply schema to the PostgreSQL database
    - Generate migration SQL with `drizzle-kit generate`; apply with `drizzle-kit migrate`; verify all tables, indexes, constraints, and enums are present; grant correct PostgreSQL role permissions (`SELECT, INSERT` only on `audit_log`; no `DELETE` on any core table for the app user) as specified in the Audit Trail section
    - _Requirements: 13.3, 16.1_

- [x] 2. Authentication and session layer
  - [x] 2.1 Configure next-auth v5 Credentials provider with PostgreSQL session storage
    - Implement `src/auth.ts`: Credentials provider that verifies `username` + `password_hash` against the `users` table using bcrypt; configure server-side session storing `{ userId, role }` via database adapter or signed iron-session cookie; add `HttpOnly`, `Secure`, `SameSite=Strict` cookie options; export `GET`/`POST` route handlers at `app/api/auth/[...nextauth]/route.ts`
    - _Requirements: 14.7, 18.1_

  - [x] 2.2 Implement auth middleware and `resolveActor` utility
    - Write `src/middleware/auth.ts` that reads the session cookie on every request and returns 401 `UNAUTHENTICATED` if no valid session; write `resolveActor(session, body)` per the design pseudocode: for `role='kiosk'` throw `ValidationError` if `guard_id` absent, else build `Actor` with `actingGuardId`; for other roles build `Actor` with only `sessionUserId`/`sessionUserRole`; export `requireRole(...roles)` RBAC middleware that maps `kiosk` → `guard` for evaluation
    - _Requirements: 14.1–14.8, 18.2–18.6_

  - [x] 2.3 Implement `/api/auth/login`, `/api/auth/logout`, `/api/auth/me` route handlers
    - POST `/api/auth/login`: delegates to next-auth sign-in, returns session user and role on success; POST `/api/auth/logout`: destroys session; GET `/api/auth/me`: returns `{ userId, role }` from current session or 401
    - _Requirements: 14.7_

  - [x] 2.4 Implement `GET /api/guards` route handler
    - Return all active users with `role = 'guard'` as `{ id, name }` list; require authenticated session (any role); used by kiosk guard picker
    - _Requirements: 18.2_

- [x] 3. Shared services: Control Number, Audit Logger, Zod DTOs
  - [x] 3.1 Implement `Control_Number_Service` with the atomic upsert algorithm
    - Create `src/services/controlNumber.ts` implementing `nextControlNumber(tx, module, year, month)` exactly per the design's SQL upsert algorithm using `INSERT … ON CONFLICT DO UPDATE … WHERE last_seq < 9999999 RETURNING last_seq`; throw `ControlNumberExhaustedError` when `RETURNING` yields no row; format result as `YYYY-MM-#######` with zero-padding; export `formatControlNumber(year, month, seq)` as a pure function for testability
    - _Requirements: 1.1, 1.2, 1.4, 1.5_

  - [x]* 3.2 Write property test for Control Number format (Property 1)
    - **Property 1: Control Number Format Correctness**
    - **Validates: Requirements 1.1**
    - Test `formatControlNumber` with `fc.integer({ min: 2020, max: 2099 })`, `fc.integer({ min: 1, max: 12 })`, `fc.integer({ min: 1, max: 9_999_999 })`: result must match `/^\d{4}-\d{2}-\d{7}$/`, year segment must equal input year, month segment must be zero-padded input month, sequence segment must be zero-padded 7-digit input seq; `numRuns: 100`

  - [x]* 3.3 Write property test for Control Number monotonic uniqueness (Property 2)
    - **Property 2: Control Number Uniqueness (Monotonic Sequence)**
    - **Validates: Requirements 1.2, 1.4**
    - Use an in-memory mock of the sequence counter; for any `N ∈ [1..100]` sequential calls to `nextControlNumber` on the same `(module, year, month)` starting from any initial `last_seq`, all N results must be distinct and each subsequent sequence component must be strictly greater than the previous; `numRuns: 100`

  - [x] 3.4 Implement `Audit_Logger` service
    - Create `src/services/auditLogger.ts` implementing `AuditLogger.log(entry, tx)` as per the design interface; write the audit entry inside the caller's transaction using the provided `tx` object; if the insert fails, re-throw so the caller's transaction rolls back; the function must never be called outside a transaction
    - _Requirements: 13.1, 13.2, 13.3, 13.6_

  - [x] 3.5 Define Zod validation schemas for all DTOs
    - Create `src/validation/` with schemas: `CreatePassSlipSchema`, `ReturnLegSchema`, `EditPassSlipSchema`, `VoidSchema` (reusable), `AttendanceEntrySchema`, `EditAttendanceSchema`, `CreateGuardReportSchema`, `EditGuardReportSchema`, `CreateEmployeeSchema`, `EditEmployeeSchema`, `DateRangeSchema` (validates `date_from <= date_to`); all field constraints from requirements (max lengths, formats, enum values, all-or-none time fields) must be encoded in the schema
    - _Requirements: 2.1–2.4, 3.1, 3.4–3.7, 8.1–8.7, 9.1–9.6, 11.1–11.3, 12.1, 12.6, 15.1_

- [x] 4. Checkpoint — Core services stable
  - Ensure all tests pass, ask the user if questions arise.

- [x] 5. Pass Slip service and API routes
  - [x] 5.1 Implement `PassSlipService.create` — outbound leg
    - Create `src/services/passSlip.ts`; implement `create(dto, actor)`: open DB transaction; validate no `control_no` in dto (reject per Req 1.3); call `nextControlNumber`; validate `type='official'` requires `destination` and `purpose`; store `position` snapshot from employee record; insert `pass_slips` row with `status='open'`, `approver_id=null`, `approval_status=null`; call `AuditLogger.log` inside same transaction; commit; return created record
    - _Requirements: 1.1, 1.3, 2.1, 2.2, 2.3, 2.5, 2.7_

  - [x]* 5.2 Write property test for client-supplied control_no rejection (Property 3)
    - **Property 3: Control Number Client Supply Rejection**
    - **Validates: Requirements 1.3, 2.5**
    - For any `control_no` value generated by `fc.oneof(fc.string(), fc.constant(''), fc.constant(null))`, calling `PassSlipService.create` with that value present in the dto must return a validation error and not create a record; `numRuns: 100`

  - [x]* 5.3 Write property test for Pass Slip creation round-trip (Property 4)
    - **Property 4: Pass Slip Creation Round-Trip**
    - **Validates: Requirements 2.1, 2.7, 1.1**
    - For any valid `CreatePassSlipDto` generated by arbitraries (date ∈ [2020-01-01, today], time_out ∈ [00:00..23:59], type in `['official','personal']`, etc.), the created record retrieved by ID must have every supplied field identical, `status='open'`, `approver_id=null`, `approval_status=null`, and `control_no` matching `/^\d{4}-\d{2}-\d{7}$/`; `numRuns: 100`

  - [x]* 5.4 Write property test for official type validation (Property 5)
    - **Property 5: Official Pass Slip Requires Destination and Purpose**
    - **Validates: Requirements 2.2, 2.3**
    - For any dto with `type='official'` where `destination` or `purpose` is null/empty/absent, `create` must reject with a validation error; for any dto with `type='personal'`, the stored record must have `destination=null` and `purpose=null` regardless of supplied values; `numRuns: 100`

  - [x] 5.5 Implement `PassSlipService.submitReturnLeg`
    - Implement `submitReturnLeg(id, dto, actor)`: reject if status ≠ `'open'`; reject if `time_in < time_out` (same-date rule); reject if `return_guard_id` is not an active user with role `'guard'` or `'admin'`; store `time_in`, `return_guard_id`, `output_on_return` (null if absent); set status to `'completed'`; run audit inside same transaction
    - _Requirements: 3.1–3.7_

  - [x]* 5.6 Write property test for return leg time ordering invariant (Property 6)
    - **Property 6: Return Leg Time Ordering Invariant**
    - **Validates: Requirements 3.1, 3.2, 3.4**
    - For any `time_in` strictly earlier than stored `time_out` on same date, `submitReturnLeg` must reject; for any `time_in >= time_out`, it must succeed, persist fields, and transition status to `'completed'`; `numRuns: 100`

  - [x]* 5.7 Write property test for return leg rejected on non-open records (Property 7)
    - **Property 7: Return Leg Rejected for Non-Open Records**
    - **Validates: Requirements 3.3**
    - For any pass slip with status in `['draft','completed','voided']`, any `submitReturnLeg` call must be rejected with an appropriate error; `numRuns: 100`

  - [x] 5.8 Implement `PassSlipService.edit` and `PassSlipService.void`
    - `edit(id, dto, actor)`: reject if status is `'voided'`; enforce admin-only for all fields except `output_on_return`/`return_guard_id`; do NOT change `control_no` when `date` changes; run audit inside same transaction; `void(id, reason, actor)`: require admin role; validate reason length 1–500; reject if already voided; transition to `'voided'`; run audit inside same transaction
    - _Requirements: 4.1–4.5, 5.1–5.7_

  - [x] 5.9 Implement `PassSlipService.search` with filtering and ordering
    - Accept `PassSlipFilters`; apply `date_from`/`date_to` (reject if from > to), `employee_name` partial match (min 2 chars), `status` filter; exclude voided by default unless status filter explicitly includes `'voided'`; order by `date DESC`, then `control_no DESC`; paginate with `page`/`page_size` (default 20, max 100); employee role scopes to own records only
    - _Requirements: 6.1–6.6, 14.8_

  - [x]* 5.10 Write property test for void semantics (Property 9, pass slip portion)
    - **Property 9: Void Semantics — Status, Reason Persistence, and Filtering (Pass Slip)**
    - **Validates: Requirements 5.1, 5.2, 5.5, 5.6**
    - For any valid void reason `fc.string({ minLength: 1, maxLength: 500 })`: after void, status must be `'voided'`, reason stored exactly; record excluded from default search; appears when `status='voided'` filter used; for reason length outside [1..500]: rejection and record unchanged; `numRuns: 100`

  - [x]* 5.11 Write property test for search result ordering (Property 10, pass slip portion)
    - **Property 10: Search Result Ordering — Pass Slips**
    - **Validates: Requirements 6.2, 6.3**
    - For any result set returned by `search`, records must be ordered by `date DESC` then `control_no DESC`; generate random sets of pass slips and verify sort invariant holds across all pages; `numRuns: 100`

  - [x]* 5.12 Write property test for invalid date range rejection (Property 11)
    - **Property 11: Invalid Date Range Rejection**
    - **Validates: Requirements 6.6, 10.6**
    - For any `date_from` strictly later than `date_to` in Pass_Slip_Module or Attendance_Module, the query must be rejected with a validation error and no DB query must execute; `numRuns: 100`

  - [x] 5.13 Wire Pass Slip API route handlers
    - Create `app/api/pass-slips/route.ts` (POST, GET), `app/api/pass-slips/[id]/route.ts` (GET, PATCH), `app/api/pass-slips/[id]/return/route.ts` (PATCH), `app/api/pass-slips/[id]/void/route.ts` (POST); apply `requireRole` guards per the role matrix; parse + validate with Zod; call service methods; return standard response envelope; handle all error codes from the taxonomy
    - _Requirements: 2.1–2.7, 3.1–3.7, 4.1–4.5, 5.1–5.7, 6.1–6.6, 14.1–14.3_

- [x] 6. Audit Logger write-audit atomicity tests
  - [x]* 6.1 Write property test for write-audit atomicity across all modules (Property 8)
    - **Property 8: Write-Audit Atomicity Across All Modules**
    - **Validates: Requirements 4.2, 9.2, 12.2, 13.1, 13.2, 13.3, 13.6**
    - For any create/edit/void on pass slip, attendance log, or guard report: simulate audit failure via a mock that throws; verify target record remains unchanged and an error is returned; for successful operations, verify exactly one new `audit_log` row exists with `before_state` matching prior snapshot (null for creates), `after_state` matching new snapshot, and `acting_user_id` equal to the correct attribution identity; `numRuns: 100`

- [x] 7. PDF Generator
  - [x] 7.1 Implement FORM7 HTML template
    - Create `src/templates/form7.html` (or a React Server Component equivalent) rendering a single A4 page with two stacked copies ("ORD's Copy" and "Guard's Copy") containing all required FORM7 fields: `control_no`, employee name, position, date, `time_out`, `time_in`, type, destination, purpose, `output_on_return`, guard on duty, immediate supervisor, employee signature line; include CSS watermark overlay using `position: absolute; transform: rotate(-45deg)` that is conditionally shown for status `'voided'` (red, opacity 0.35) and `'draft'` (grey, opacity 0.25) and hidden for `'open'`/`'completed'`
    - _Requirements: 7.1–7.5_

  - [x]* 7.2 Write property test for PDF template rendering (Property 12)
    - **Property 12: PDF Template Renders All Required Fields and Correct Watermarks**
    - **Validates: Requirements 7.1, 7.2, 7.3, 7.4, 7.5**
    - For any `PassSlip` object generated by arbitrary, the rendered HTML string from `renderPassSlipTemplate(passSlip)` must contain each required field value as a substring, contain both "ORD's Copy" and "Guard's Copy" labels, contain "VOID" when `status='voided'`, contain "DRAFT" when `status='draft'`, and contain neither when `status='open'` or `'completed'`; `numRuns: 100`

  - [x] 7.3 Implement `PDF_Generator` service using Puppeteer
    - Create `src/services/pdfGenerator.ts` implementing `renderPassSlip(passSlip)`: launch (or reuse) headless Chromium; set HTML content from template; call `page.pdf({ format: 'A4' })`; return `Buffer`; catch any Puppeteer error and throw `PdfGenerationError`; do NOT stream partial bytes; implement a startup health check that verifies Puppeteer can launch
    - _Requirements: 7.6, 7.7, 7.8_

  - [x] 7.4 Wire `GET /api/pass-slips/[id]/pdf` route handler
    - Verify authenticated session and role (`guard`, `supervisor`, `admin`); fetch pass slip; call `pdfGenerator.renderPassSlip`; stream bytes as `application/pdf` with `Content-Disposition: attachment`; on `PdfGenerationError` return HTTP 500 with JSON error body
    - _Requirements: 7.1, 7.8_

- [x] 8. Checkpoint — Pass Slip module complete
  - Ensure all tests pass, ask the user if questions arise.

- [x] 9. Attendance Logbook service and API routes
  - [x] 9.1 Implement `AttendanceService.upsert`
    - Create `src/services/attendance.ts`; implement `upsert(dto, actor)`: validate `employee_id` exists in `employees`; validate all-or-none time fields (reject if 1, 2, or 3 of 4 provided); validate `employee_type` is a known enum value; run INSERT with `ON CONFLICT (employee_id, date) DO UPDATE` inside a transaction; call `AuditLogger.log` (operation `'create'` for new row, `'edit'` for update with before-state); commit; return upserted record
    - _Requirements: 8.1–8.7_

  - [x]* 9.2 Write property test for attendance upsert uniqueness (Property 13)
    - **Property 13: Attendance Log Upsert Uniqueness**
    - **Validates: Requirements 8.2**
    - For any N ≥ 1 upsert calls with the same `(employee_id, date)` pair using `fc.array(attendanceDto, { minLength: 1, maxLength: 20 })`, the total non-voided row count for that pair must be exactly 1 after all N calls, and the stored row must reflect the most recent upsert values; `numRuns: 100`

  - [x]* 9.3 Write property test for all-or-none time fields (Property 14)
    - **Property 14: Attendance All-or-None Time Fields**
    - **Validates: Requirements 8.6**
    - For any dto with exactly 1, 2, or 3 of `(am_in, am_out, pm_in, pm_out)` non-null (generated via `fc.subarray` of the four field names with `{ minLength: 1, maxLength: 3 }`), `upsert` must reject with a validation error; for all-four or all-null submissions, must accept; `numRuns: 100`

  - [x] 9.4 Implement `AttendanceService.edit` and `AttendanceService.void`
    - `edit(id, dto, actor)`: reject if voided; if actor role is `'guard'`, reject if `date < today`; persist updated editable fields; run audit inside same transaction; `void(id, reason, actor)`: admin only; validate reason 1–500 chars; reject if already voided; mark voided; run audit
    - _Requirements: 9.1–9.7_

  - [x]* 9.5 Write property test for void semantics (Property 9, attendance portion)
    - **Property 9: Void Semantics — Status, Reason Persistence, and Filtering (Attendance)**
    - **Validates: Requirements 9.3, 9.4, 16.2, 16.3**
    - Same invariant as 5.10 but over `AttendanceService`: valid reason → voided, stored exactly, excluded from default filter; invalid reason → rejected unchanged; `numRuns: 100`

  - [x] 9.6 Implement `AttendanceService.search` and `AttendanceService.export`
    - `search`: filters `date_from`/`date_to` (reject if from > to), `employee_name` partial match, `employee_type`; exclude voided by default; order `date ASC` then employee name ASC; `export(filters, format, actor)`: admin/supervisor only; same filters; produce XLSX or CSV via ExcelJS with required columns and headers in first row; return `Buffer`
    - _Requirements: 10.1–10.6_

  - [x]* 9.7 Write property test for attendance search ordering (Property 10, attendance portion)
    - **Property 10: Search Result Ordering — Attendance**
    - **Validates: Requirements 10.2**
    - For any result set from `AttendanceService.search`, records must be ordered by `date ASC` then employee name ASC; `numRuns: 100`

  - [x] 9.8 Wire Attendance API route handlers
    - Create `app/api/attendance/route.ts` (POST, GET), `app/api/attendance/[id]/route.ts` (PATCH), `app/api/attendance/[id]/void/route.ts` (POST), `app/api/attendance/export/route.ts` (GET); apply `requireRole` per role matrix; validate with Zod; call service methods; return standard envelope; set correct `Content-Type` and `Content-Disposition` for export responses
    - _Requirements: 8.1–8.7, 9.1–9.7, 10.1–10.6, 14.4, 14.5_

- [x] 10. Guard Report service and API routes
  - [x] 10.1 Implement `GuardReportService.create`
    - Create `src/services/guardReport.ts`; implement `create(dto, actor)`: validate `guard_id` references active user with `role='guard'`; validate `period_end >= period_start`; query for existing non-voided reports for same `guard_id` where periods overlap (`period_start_2 <= period_end_1 AND period_start_1 <= period_end_2`); if overlap found, reject with `CONFLICT` including existing report's ID; insert; run audit
    - _Requirements: 11.1–11.6_

  - [x]* 10.2 Write property test for guard report period non-overlap (Property 15)
    - **Property 15: Guard Report Period Non-Overlap**
    - **Validates: Requirements 11.4, 11.5**
    - For any two report submissions with the same `guard_id` where periods overlap (using `fc.date` arbitraries), the second must be rejected with a conflict error containing the first report's ID; for non-overlapping periods, both must be accepted; verify boundary: `[1,10]` and `[11,20]` accepted, `[1,10]` and `[10,20]` rejected; `numRuns: 100`

  - [x] 10.3 Implement `GuardReportService.edit` and `GuardReportService.void`
    - `edit(id, dto, actor)`: for guard role, require `guard_id` matches acting user; reject if voided; validate period constraints; run audit; `void(id, reason, actor)`: admin only; validate reason 1–500; reject if already voided; run audit; neither operation permanently deletes data
    - _Requirements: 12.1–12.7_

  - [x]* 10.4 Write property test for void semantics (Property 9, guard report portion)
    - **Property 9: Void Semantics — Status, Reason Persistence, and Filtering (Guard Report)**
    - **Validates: Requirements 12.5, 12.6, 16.2, 16.3**
    - Same void invariant applied to `GuardReportService`: valid reason → voided and stored; invalid reason → rejected unchanged; `numRuns: 100`

  - [x] 10.5 Wire Guard Report API route handlers
    - Create `app/api/guard-reports/route.ts` (POST, GET), `app/api/guard-reports/[id]/route.ts` (GET, PATCH), `app/api/guard-reports/[id]/void/route.ts` (POST); apply `requireRole` per role matrix (guard can file/edit own, admin sees all); validate with Zod; return standard envelope
    - _Requirements: 11.1–11.6, 12.1–12.7, 14.6_

- [x] 11. Employee master list and User management
  - [x] 11.1 Implement Employee service and API routes
    - Create `src/services/employee.ts` with `create`, `edit`, `void`, `search`; validate `name`/`position`/`division` 1–100 chars; `void` requires reason 1–500 chars; no hard deletes (reject any operation attempting permanent delete); on void, retain all historical pass slip and attendance references; run audit for create/edit/void; wire `app/api/employees/` routes (POST, GET, PATCH `[id]`, POST `[id]/void`) with admin-only auth
    - _Requirements: 15.1–15.6, 16.1–16.4_

  - [x] 11.2 Implement User management API routes
    - Wire `app/api/users/` (POST, GET, PATCH `[id]`) routes with admin-only `requireRole`; validate `name`, `role`, `is_active` fields; hash passwords with bcrypt before storing; no password retrieval endpoint
    - _Requirements: 14.7_

  - [x] 11.3 Implement Audit Trail read endpoint
    - Wire `GET /api/audit/[table]/[recordId]` route: admin only; query `audit_log` for all entries matching `(table_name, record_id)` ordered by `created_at ASC`; return full entries including `before_state`, `after_state`, `acting_user_id`, `acting_guard_id`, timestamp
    - _Requirements: 13.4_

- [x] 12. Kiosk attribution invariant tests
  - [x]* 12.1 Write property test for kiosk attribution invariant (Property 16)
    - **Property 16: Kiosk Attribution Invariant**
    - **Validates: Requirements 18.2, 18.5, 18.6**
    - For any action submitted with a kiosk session actor where `actingGuardId` is a valid guard UUID, the stored `outbound_guard_id`/`return_guard_id`/`recorded_by` field must equal `actingGuardId`; the `audit_log` entry's `acting_user_id` must equal `actingGuardId`; the kiosk account UUID must not appear in any of these fields; for kiosk sessions without `guard_id`, the service must reject with `VALIDATION_ERROR`; `numRuns: 100`

- [x] 13. Checkpoint — All services and API routes complete
  - Ensure all tests pass, ask the user if questions arise.

- [x] 14. UI pages — Authentication and shell
  - [x] 14.1 Implement login page and authenticated layout shell
    - Create `app/login/page.tsx`: username/password form, POST to next-auth, redirect to `/` on success, show field-level errors on failure; create root authenticated layout `app/(app)/layout.tsx`: verify session (redirect to `/login` if absent), render nav sidebar with role-aware links (guard sees pass slips + attendance + guard reports; admin sees all + user/employee management + audit trail); add skip-navigation link and accessible landmark roles
    - _Requirements: 14.7, 18.1_

- [x] 15. UI pages — Pass Slips
  - [x] 15.1 Implement Pass Slip list/search page
    - Create `app/(app)/pass-slips/page.tsx`: search form with `date_from`, `date_to`, employee name, status filter; GET `/api/pass-slips` with query params; render table with control number, employee name, date, time_out, type, status columns; paginated results; "no results" message with filter preservation; "(Inactive)" label for voided employees; link to detail view and "Record Return" for open records
    - _Requirements: 6.1–6.6, 15.5_

  - [x] 15.2 Implement Pass Slip create page (outbound leg)
    - Create `app/(app)/pass-slips/new/page.tsx`: kiosk guard picker (shown when `role='kiosk'`); employee searchable selector; date input defaulting to today; time_out defaulting to current HH:MM; type radio (Official/Personal); conditional destination+purpose fields for Official; immediate supervisor name free text; POST `/api/pass-slips`; on success redirect to detail with control number banner; on error show field-level messages; all inputs have associated `<label>` and `aria-describedby` for errors
    - _Requirements: 2.1–2.7, 1.6, 18.2–18.4_

  - [x] 15.3 Implement Pass Slip detail page and return leg form
    - Create `app/(app)/pass-slips/[id]/page.tsx`: display all pass slip fields; "Record Return" button (guard/admin, only when `status='open'`): inline or modal form with time_in, return guard, output on return; PATCH `/api/pass-slips/[id]/return`; "Edit" button (admin only, not voided); "Void" button (admin only, not voided) opens modal with void reason textarea; "Print PDF" button opens `GET /api/pass-slips/[id]/pdf`; "View History" tab calls `GET /api/audit/pass_slips/[id]` and renders timeline; focus management after successful submission
    - _Requirements: 3.1–3.7, 4.1–4.5, 5.1–5.7, 7.1, 13.4_

- [x] 16. UI pages — Attendance Logbook
  - [x] 16.1 Implement Attendance entry page
    - Create `app/(app)/attendance/new/page.tsx` (or date-scoped `/attendance/date/[date]/page.tsx`): kiosk guard picker; date selector defaulting to today; employee list with per-row fields: employee type selector, am_in/am_out/pm_in/pm_out time inputs (all-or-none hint), remarks; save per row via POST `/api/attendance`; inline row-level validation errors; `aria-label` on time inputs
    - _Requirements: 8.1–8.7, 18.2–18.4_

  - [x] 16.2 Implement Attendance filter/list and export page
    - Create `app/(app)/attendance/page.tsx`: filters for date range, employee name, employee_type; GET `/api/attendance`; table with date, employee name, type, time columns, remarks, recorded_by; "Export CSV" / "Export XLSX" buttons call `GET /api/attendance/export?format=csv|xlsx`; `date_from > date_to` shows inline error before submitting
    - _Requirements: 10.1–10.6_

- [x] 17. UI pages — Guard Reports
  - [x] 17.1 Implement Guard Report list and create page
    - Create `app/(app)/guard-reports/page.tsx`: list own reports (guard) or all (admin); create form at `/guard-reports/new`: guard selector (pre-populated for guard role, editable for admin), period start/end date pickers, narrative textarea with 10,000 char counter; POST `/api/guard-reports`; on conflict show error with link to conflicting report; edit/void from detail page
    - _Requirements: 11.1–11.6, 12.1–12.7_

- [x] 18. UI pages — Admin: Employee and User Management
  - [x] 18.1 Implement Employee management pages
    - `app/(app)/admin/employees/page.tsx`: searchable employee list with "(Inactive)" badges; create form; edit form (same fields); void button with reason modal; all admin-only; field-level validation errors for each constraint; accessible form structure
    - _Requirements: 15.1–15.6_

  - [x] 18.2 Implement User management page
    - `app/(app)/admin/users/page.tsx`: list all users with name, role, active status; create user form (name, username, password, role); edit form (name, role, is_active — no password field on edit); admin-only; no delete button (no hard-delete UI)
    - _Requirements: 14.7, 16.1_

- [x] 19. Kiosk dashboard
  - [x] 19.1 Implement kiosk-specific dashboard and guard picker component
    - Create `app/(app)/kiosk/dashboard/page.tsx`: landing page for `role='kiosk'` session showing "Record Pass Slip" and "Record Attendance" actions; create reusable `<GuardPicker>` component that fetches `/api/guards`, renders accessible dropdown, and passes selected `guard_id` to child form via context or prop; component blocks form submission until a guard is selected; keyboard-navigable
    - _Requirements: 18.1–18.6_

- [x] 20. Final checkpoint — All tests pass, feature complete
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP delivery, but all 16 correctness properties are recommended for production quality.
- The design uses TypeScript throughout — all code examples and implementations should be TypeScript.
- Property tests use `fast-check` with `numRuns: 100` minimum; tag each test comment with `Feature: pass-slip-attendance-logbook, Property N: <text>`.
- Every write operation (create/edit/void) must call `AuditLogger.log` inside the same DB transaction — this is non-negotiable and is enforced by passing the `tx` object.
- The kiosk account's UUID must never appear in `outbound_guard_id`, `return_guard_id`, `recorded_by`, or `audit_log.acting_user_id` — always resolve attribution via `actor.actingGuardId ?? actor.sessionUserId`.
- No hard deletes anywhere; any attempt to permanently delete a record must return an error.
- Control numbers are server-assigned; any client-supplied `control_no` in a creation request must be rejected regardless of value.
- All `date` fields accept back-dates to 2020-01-01 to support future migration.
- No bulk import UI or API is in scope; defer to the Data Migration spec.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2"] },
    { "id": 2, "tasks": ["1.3"] },
    { "id": 3, "tasks": ["2.1", "3.5"] },
    { "id": 4, "tasks": ["2.2", "3.1", "3.4"] },
    { "id": 5, "tasks": ["2.3", "2.4", "3.2", "3.3"] },
    { "id": 6, "tasks": ["5.1", "9.1", "10.1"] },
    { "id": 7, "tasks": ["5.2", "5.3", "5.4", "5.5", "9.2", "9.3", "10.2", "11.1"] },
    { "id": 8, "tasks": ["5.6", "5.7", "5.8", "5.9", "6.1", "9.4", "9.6", "10.3", "11.2", "11.3"] },
    { "id": 9, "tasks": ["5.10", "5.11", "5.12", "7.1", "9.5", "9.7", "10.4", "12.1"] },
    { "id": 10, "tasks": ["5.13", "7.2", "9.8", "10.5"] },
    { "id": 11, "tasks": ["7.3"] },
    { "id": 12, "tasks": ["7.4"] },
    { "id": 13, "tasks": ["14.1"] },
    { "id": 14, "tasks": ["15.1", "15.2", "16.1", "17.1", "18.1", "18.2", "19.1"] },
    { "id": 15, "tasks": ["15.3", "16.2"] }
  ]
}
```

# Post-Implementation Fixes

- [ ] 21. Fix kiosk authentication issues
  - [ ] 21.1 Resolve authentication required error when saving forms from kiosk
    - Investigate and fix the "authentication required" error occurring when users save forms from kiosk mode; ensure kiosk users can submit attendance and pass slip forms without authentication prompts; verify guest/public access works properly for kiosk functionality; ensure forms are properly saved to database from kiosk mode
    - _Requirements: 18.1, 18.2, 14.1_

- [ ] 22. Implement admin record viewing
  - [ ] 22.1 Enable admin to view all submitted records
    - Implement admin interface to view all submitted attendance and pass slip records; ensure records display all submitted information clearly in organized format; admin should have access to comprehensive record listings for both modules
    - _Requirements: 6.1, 10.1, 14.4, 14.5_

- [ ] 23. Add print functionality for records
  - [ ] 23.1 Implement print capability for attendance and pass slip records
    - Add print functionality allowing admin to print individual attendance and pass slip records; ensure print format matches official form layouts; verify print functionality works across different browsers with proper formatting; include print preview functionality
    - _Requirements: 7.1, 7.8_