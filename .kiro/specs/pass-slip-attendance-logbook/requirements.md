# Requirements Document

## Introduction

The Pass Slip & Attendance Logbook module is a web-based administrative records system for the DOST Regional Office, replacing an existing Excel/VBA workbook. It covers three sub-features:

1. **Pass Slip (FORM7)** — records an employee leaving and returning to office premises, with outbound and inbound legs logged by the security guard on duty.
2. **Attendance Logbook** — daily AM/PM time-in/time-out entries recorded by the guard on duty for Permanent, Contractual, and OJT employees.
3. **Guard Duty Accomplishment Report** — a lightweight monthly narrative report filed by guards.

The system runs on on-premises government infrastructure. No cloud services or third-party integrations are assumed. User and role management is self-contained. Historical data migration (from 2020–2026 Excel sheets) is flagged as a dependency for a separate future spec and is explicitly **out of scope** for this module.

---

## Glossary

- **Pass_Slip_Module**: The subsystem responsible for creating, updating, voiding, and exporting Pass Slip records.
- **Attendance_Module**: The subsystem responsible for recording and exporting daily attendance log entries.
- **Guard_Report_Module**: The subsystem responsible for creating and viewing Guard Duty Accomplishment Reports.
- **Control_Number_Service**: The server-side service that generates collision-free control numbers.
- **PDF_Generator**: The server-side component that renders printable PDF documents matching official form layouts.
- **Audit_Logger**: The component that records the full change history (before/after values) for every create, edit, or void operation, attributed to the acting user.
- **Employee**: A person listed in the employees master table who may hold a Pass Slip or appear in the Attendance Logbook.
- **Guard**: A user with role `guard` who records Pass Slip entries and Attendance Logbook entries on behalf of employees.
- **Admin**: A user with role `admin` who manages the employee master list, user accounts, and system-wide settings.
- **Supervisor**: A user with role `supervisor` whose name may appear on a Pass Slip as an optional approver; no system-enforced approval workflow exists in v1.
- **Control_Number**: A unique identifier with format `YYYY-MM-#######` (e.g., `2026-07-0000744`), generated per module per calendar month, padded to 7 digits.
- **Pass_Slip_Status**: One of `draft`, `open` (outbound leg filed, return not yet recorded), `completed` (return leg filed), `voided`.
- **Employee_Type**: One of `permanent`, `contractual`, `ojt`.
- **Outbound_Leg**: The first half of a Pass Slip — fields recorded when the employee leaves.
- **Return_Leg**: The second half of a Pass Slip — fields recorded when the employee returns.
- **Void**: A soft-delete operation that marks a record as inactive with a mandatory reason, preserving the original data.
- **Audit_Trail**: An append-only log of all state changes to a record, including the before-state, after-state, acting user, and timestamp.

---

## Open Items / Flagged Assumptions

The following items require stakeholder decisions before or during design. They are surfaced here rather than assumed away.

### OI-1: Guard Login Model (RESOLVED)
> **Decision:** A single shared kiosk credential is used to authenticate the guard workstation session. However, every action that requires guard attribution (Pass Slip outbound/return, Attendance Logbook entry) must prompt the operator to select or confirm the specific named guard from the employee/users list before the action is accepted. The selected guard's ID is stored in `outbound_guard_id`, `return_guard_id`, and `recorded_by` fields for audit purposes. The shared kiosk account itself is never used as the attribution identity for individual actions — only for the session login.

### OI-2: Employee Self-Service Scope
> **Assumption made (flag for confirmation):** Employees with role `employee` may file the Outbound_Leg of their own Pass Slip (time_out, type, destination, purpose). The Return_Leg (output_on_return, time_in, return_guard_id) must be recorded by a Guard or Admin, because physical return verification belongs to the guard station.

### OI-3: Supervisor Approval Gate (v1 decision confirmed)
> **Confirmed:** No mandatory supervisor approval workflow in v1. The `immediate_supervisor_name` field is stored as a free-text string for printing on the form only. An optional `approver_id` (FK to users) and `approval_status` field are included in the schema so the approval gate can be activated in a future version without a schema migration.

### OI-4: Historical Data Migration
> **Confirmed out of scope.** The 2020–2026 Excel sheets will be handled by a separate "Data Migration" spec. This spec MUST NOT include migration tasks or an import UI.

### OI-5: Authentication Mechanism
> **Assumption made (flag for confirmation):** Authentication uses username/password with server-side session management (no external OAuth or LDAP in v1), appropriate for on-premises government infrastructure.

---

## Requirements

---

### Requirement 1: Control Number Generation

**User Story:** As a Guard or Employee filing a Pass Slip, I want a unique control number automatically assigned to each record, so that records are unambiguously identifiable and match the existing paper form numbering scheme.

#### Acceptance Criteria

1. WHEN a Pass Slip record is created, THE Control_Number_Service SHALL assign a Control_Number with the format `YYYY-MM-#######`, where `YYYY` is the four-digit year, `MM` is the zero-padded two-digit month of the record date, and `#######` is a zero-padded 7-digit sequential counter starting at `0000001` and scoped to that year-month combination.
2. WHEN two or more Pass Slip creation requests are received concurrently for the same year-month, THE Control_Number_Service SHALL assign distinct Control_Numbers to each request without collision.
3. IF a Pass Slip creation request payload includes a `control_no` field — regardless of whether the value is non-empty, empty, or null — THEN THE Pass_Slip_Module SHALL reject the request and return an error indicating that control numbers are system-assigned and may not be provided by the client, without creating the record.
4. IF a Pass Slip creation transaction fails after a Control_Number has been reserved, THEN THE Control_Number_Service SHALL NOT reuse that Control_Number for a subsequent successful creation in the same year-month sequence.
5. IF the sequential counter for a given year-month reaches `9999999`, THEN THE Control_Number_Service SHALL reject any further Pass Slip creation requests for that year-month and return an error indicating the control number space for that period is exhausted.
6. WHEN a Pass Slip record is successfully created, THE Pass_Slip_Module SHALL display the assigned Control_Number to the submitting user within 2 seconds of the successful response.

---

### Requirement 2: Pass Slip — Outbound Leg Filing

**User Story:** As an Employee or Guard, I want to file the outbound half of a Pass Slip when an employee leaves the premises, so that the departure is formally recorded.

#### Acceptance Criteria

1. WHEN an authenticated user with role `employee`, `guard`, or `admin` submits a new Pass Slip, THE Pass_Slip_Module SHALL create a Pass Slip record with status `open` and persist the following fields: `control_no` (system-assigned), `employee_id`, `position`, `date` (format `YYYY-MM-DD`), `time_out` (format `HH:MM`, 24-hour), `outbound_guard_id`, `type` (`official` or `personal`), and `immediate_supervisor_name` (free text, max 100 characters).
2. IF the Pass Slip `type` is `official`, THEN THE Pass_Slip_Module SHALL require non-empty values for both `destination` and `purpose` (each max 500 characters) before accepting the submission.
3. WHEN the Pass Slip `type` is `personal`, THE Pass_Slip_Module SHALL store `destination` and `purpose` as null and SHALL NOT require them.
4. IF a required field (`employee_id`, `date`, `time_out`, `outbound_guard_id`, `type`) is absent or invalid on submission, THEN THE Pass_Slip_Module SHALL reject the request and return a validation error identifying each missing or invalid field by name.
5. IF a submission for a new Pass Slip includes a `control_no` field, THEN THE Pass_Slip_Module SHALL ignore the supplied value and assign a system-generated Control_Number per Requirement 1.
6. IF a Return_Leg submission is received for a Pass Slip whose status is `open` and the Outbound_Leg has already been filed, THEN THE Pass_Slip_Module SHALL reject any attempt to re-submit an Outbound_Leg for that same record and return an error indicating the record is not accepting a new outbound entry.
7. WHEN a new Pass Slip record is created, THE Pass_Slip_Module SHALL record the `approver_id` and `approval_status` fields as null; WHEN these fields are null, THE Pass_Slip_Module SHALL treat the Pass Slip as not subject to approval gating.

---

### Requirement 3: Pass Slip — Return Leg Filing

**User Story:** As a Guard or Admin, I want to record an employee's return by completing the Return Leg of an existing Pass Slip, so that the full departure-and-return cycle is captured.

#### Acceptance Criteria

1. WHEN an authenticated user with role `guard` or `admin` submits a Return_Leg update for a Pass Slip with status `open`, THE Pass_Slip_Module SHALL persist `time_in`, `return_guard_id`, and `output_on_return`.
2. WHEN the Return_Leg fields are successfully persisted, THE Pass_Slip_Module SHALL transition the Pass Slip status from `open` to `completed` and return a confirmation response to the submitting user.
3. IF a Return_Leg submission is received for a Pass Slip whose status is not `open`, THEN THE Pass_Slip_Module SHALL reject the request and return an error stating that the record is not awaiting a return entry.
4. IF `time_in` supplied in the Return_Leg is earlier than the `time_out` recorded in the Outbound_Leg on the same date, THEN THE Pass_Slip_Module SHALL reject the submission and return an error indicating that the return time cannot precede the departure time.
5. IF `output_on_return` is absent on Return_Leg submission, THEN THE Pass_Slip_Module SHALL accept the submission with `output_on_return` stored as null and SHALL NOT treat `output_on_return` as a mandatory field.
6. IF a Return_Leg submission is received from a user whose role is not `guard` or `admin`, THEN THE Pass_Slip_Module SHALL reject the submission and return an authorization error indicating the user's role does not permit filing a Return_Leg.
7. IF `return_guard_id` supplied in the Return_Leg does not reference an existing user account with role `guard` or `admin`, THEN THE Pass_Slip_Module SHALL reject the submission and return a validation error identifying the unrecognized `return_guard_id`.

---

### Requirement 4: Pass Slip — Editing

**User Story:** As an Admin, I want to correct errors in a Pass Slip record after filing, so that the official record reflects accurate information.

#### Acceptance Criteria

1. WHEN an authenticated user with role `admin` submits an edit to a Pass Slip with status `open` or `completed`, THE Pass_Slip_Module SHALL persist the updated field values.
2. WHEN the updated values are persisted, THE Pass_Slip_Module SHALL instruct THE Audit_Logger to record the prior field values, the new field values, the acting user's ID, and the UTC timestamp; IF THE Audit_Logger fails to persist the audit entry, THEN THE Pass_Slip_Module SHALL roll back the field update and return an error to the caller notifying the user that their changes were not saved.
3. IF an edit is submitted for a Pass Slip with status `voided`, THEN THE Pass_Slip_Module SHALL reject the edit and return an error stating that voided records cannot be modified.
4. WHEN an edit changes the `date` field of a Pass Slip, THE Pass_Slip_Module SHALL NOT alter the already-assigned Control_Number.
5. IF a user with role `guard` or `employee` attempts to edit any field other than `output_on_return` or `return_guard_id` on a Pass Slip with status `open` or `completed`, THEN THE Pass_Slip_Module SHALL reject the edit and return an authorization error.

---

### Requirement 5: Pass Slip — Void (Soft Delete)

**User Story:** As an Admin, I want to void a Pass Slip record rather than permanently deleting it, so that the audit trail is preserved and the record remains searchable for reference.

#### Acceptance Criteria

1. WHEN an authenticated user with role `admin` submits a void request for a Pass Slip with status `open` or `completed`, accompanied by a non-empty `void_reason` of 1–500 characters, THE Pass_Slip_Module SHALL transition the Pass Slip status to `voided` and persist the `void_reason`.
2. IF a void request for a Pass Slip fails for any reason — including absent or invalid `void_reason`, permission changes, or system errors — THEN THE Pass_Slip_Module SHALL return an error indicating the failure reason and leave the record unchanged.
3. THE Pass_Slip_Module SHALL NOT permanently delete any Pass Slip record; all void operations SHALL be soft-deletes preserving all field values.
4. WHEN a Pass Slip is voided, THE Audit_Logger SHALL record the voiding action, the `void_reason`, the acting user's ID, and the UTC timestamp.
5. IF a search query is submitted without a status filter, THEN THE Pass_Slip_Module SHALL exclude Pass Slip records with status `voided` from the results.
6. IF a search query is submitted with a status filter that explicitly includes `voided`, THEN THE Pass_Slip_Module SHALL include records with status `voided` in the results.
7. IF a void request is submitted for a Pass Slip that already has status `voided`, THEN THE Pass_Slip_Module SHALL reject the request and return an error indicating the record is already voided, leaving it unchanged.

---

### Requirement 6: Pass Slip — Search and Filter

**User Story:** As a Guard, Admin, or Supervisor, I want to search and filter Pass Slip records, so that I can quickly locate specific records for review or follow-up.

#### Acceptance Criteria

1. THE Pass_Slip_Module SHALL provide a search interface that accepts, individually or in combination: date range (`date_from`, `date_to`, both inclusive), employee name (partial match, minimum 2 characters), and Pass Slip status (`open`, `completed`, `voided`, or all).
2. WHEN a search query is submitted, THE Pass_Slip_Module SHALL return all Pass Slip records matching all supplied filter criteria, ordered by `date` descending then by `control_no` descending.
3. WHEN no filter criteria are supplied, THE Pass_Slip_Module SHALL return all non-voided Pass Slip records ordered by `date` descending then by `control_no` descending.
4. WHEN a search returns one or more results, THE Pass_Slip_Module SHALL display at minimum: Control_Number, employee name, date, time_out, type, and status for each record.
5. WHEN a search returns no results, THE Pass_Slip_Module SHALL display a no-results message and preserve the submitted filter criteria in the search form.
6. IF a search query is submitted with `date_from` later than `date_to`, THEN THE Pass_Slip_Module SHALL reject the query, return an error indicating the date range is invalid, and perform no search.

---

### Requirement 7: Pass Slip — PDF Export

**User Story:** As a Guard or Admin, I want to export a finalized Pass Slip as a PDF, so that I can print the two-copy paper form required by the current physical workflow.

#### Acceptance Criteria

1. WHEN an authenticated user with role `guard`, `admin`, or `supervisor` requests a PDF export of a Pass Slip with status `completed` or `open`, THE PDF_Generator SHALL produce a single-page PDF containing two identical stacked copies of the Pass Slip, labeled "ORD's Copy" (top) and "Guard's Copy" (bottom).
2. THE PDF_Generator SHALL render the PDF to match the official FORM7 layout, including the agency letterhead, form title, employee name, position, date, `time_out`, `time_in`, `type`, `destination`, `purpose`, `output_on_return`, `outbound_guard_id`, `return_guard_id`, `immediate_supervisor_name`, and signature lines for the employee, the guard on duty, and the immediate supervisor.
3. WHEN `approver_id` and `approval_status` are null, THE PDF_Generator SHALL render the supervisor signature line as blank (not hidden), preserving the form layout.
4. IF a PDF export is requested for a Pass Slip with status `voided`, THEN THE PDF_Generator SHALL include a "VOID" watermark rendered diagonally across each copy at an opacity sufficient to be legible without obscuring the form fields, and SHALL NOT suppress the record data.
5. IF a PDF export is requested for a Pass Slip with status `draft`, THEN THE PDF_Generator SHALL include a "DRAFT" watermark rendered diagonally across each copy at an opacity sufficient to be legible without obscuring the form fields.
6. THE PDF_Generator SHALL generate PDFs server-side.
7. THE Pass_Slip_Module SHALL NOT use client-side rendering for official form output.
8. IF the PDF_Generator encounters an error during generation, THEN THE Pass_Slip_Module SHALL return an error indication to the user and SHALL NOT deliver a partial or empty file.

---

### Requirement 8: Attendance Logbook — Daily Entry

**User Story:** As a Guard, I want to record the daily AM and PM time-in/time-out entries for each employee, so that an accurate official attendance record is maintained.

#### Acceptance Criteria

1. WHEN an authenticated user with role `guard` or `admin` submits an attendance log entry, THE Attendance_Module SHALL persist the following fields: `employee_id`, `employee_type` (`permanent`, `contractual`, or `ojt`), `date` (format `YYYY-MM-DD`), `am_in` (format `HH:MM`, 24-hour), `am_out` (format `HH:MM`, 24-hour), `pm_in` (format `HH:MM`, 24-hour), `pm_out` (format `HH:MM`, 24-hour), `remarks` (free text, max 500 characters), and `recorded_by`.
2. IF an attendance entry already exists for the same `employee_id` and `date`, THEN THE Attendance_Module SHALL update the existing record rather than creating a duplicate, and SHALL instruct THE Audit_Logger to record the prior and new field values with the acting user's ID and UTC timestamp.
3. IF a required field (`employee_id`, `employee_type`, `date`, `recorded_by`) is absent or invalid on submission — including a malformed date, an unrecognized `employee_type` value, or a time value outside the `HH:MM` 24-hour range — THEN THE Attendance_Module SHALL reject the request and return a validation error identifying each failing field by name and reason.
4. WHEN an attendance log entry is submitted with all four time fields (`am_in`, `am_out`, `pm_in`, `pm_out`) null or absent, THE Attendance_Module SHALL accept the entry as a valid absent-day record with the provided `remarks`.
5. THE Attendance_Module SHALL NOT enforce that an employee's `employee_type` tag on a given attendance entry matches any prior entry; the tag on each entry is independent and reflects the employee's classification on that date.
6. IF a submission provides some but not all of the four time fields (`am_in`, `am_out`, `pm_in`, `pm_out`), THEN THE Attendance_Module SHALL reject the request and return a validation error requiring either all four time fields or none.
7. IF `employee_id` supplied in the submission does not match a known employee record, THEN THE Attendance_Module SHALL reject the request and return a validation error indicating the unrecognized `employee_id`.

---

### Requirement 9: Attendance Logbook — Editing and Void

**User Story:** As an Admin, I want to correct or void attendance log entries, so that errors are fixed and the record history is preserved.

#### Acceptance Criteria

1. WHEN an authenticated user with role `admin` submits an edit to a non-voided attendance log entry, THE Attendance_Module SHALL persist the updated values for editable fields (`am_in`, `am_out`, `pm_in`, `pm_out`, `date`, `remarks`).
2. WHEN the updated values are persisted, THE Attendance_Module SHALL instruct THE Audit_Logger to record the before-state, after-state, acting user's ID, and UTC timestamp; IF THE Audit_Logger fails to persist the audit entry, THEN THE Attendance_Module SHALL roll back the field update and return an error to the caller.
3. WHEN an authenticated user with role `admin` submits a void request for a non-voided attendance log entry with a `void_reason` of 1–500 characters, THE Attendance_Module SHALL mark the entry as voided and persist the `void_reason`.
4. IF a void request is submitted without a `void_reason`, or with a `void_reason` that is empty or exceeds 500 characters, THEN THE Attendance_Module SHALL reject the request and return an error describing the `void_reason` requirement, leaving the entry unchanged.
5. THE Attendance_Module SHALL NOT permanently delete any attendance log entry.
6. IF an edit or void request is submitted for an attendance log entry that is already voided, THEN THE Attendance_Module SHALL reject the request and return an error stating that voided entries cannot be modified.
7. IF a user with role `guard` attempts to edit an attendance log entry where `date` is earlier than the current calendar date, THEN THE Attendance_Module SHALL reject the edit and return an authorization error directing the user to contact an Admin.

---

### Requirement 10: Attendance Logbook — Search and Export

**User Story:** As an Admin or HR officer, I want to search, filter, and export attendance records by date range and by employee, so that I can produce HR and payroll reference reports.

#### Acceptance Criteria

1. THE Attendance_Module SHALL provide a filter interface accepting: date range (`date_from`, `date_to`, both inclusive), employee name (partial match), and `employee_type` (`permanent`, `contractual`, `ojt`, or all).
2. WHEN a filter query is submitted, THE Attendance_Module SHALL return all matching non-voided attendance log entries ordered by `date` ascending then by employee name ascending.
3. WHEN an export is requested for a filtered result set, THE Attendance_Module SHALL produce a downloadable file containing, at minimum, the columns: `date`, `employee_name`, `employee_type`, `am_in`, `am_out`, `pm_in`, `pm_out`, `remarks`, and `recorded_by` for all matching records.
4. WHEN an export is requested, THE Attendance_Module SHALL include column headers in the first row of the exported file.
5. WHEN no export format other than CSV is configured by an Admin, THE Attendance_Module SHALL produce the export as a CSV file; WHERE an alternative export format (e.g., XLSX) is configured by an Admin, THE Attendance_Module SHALL produce the export in that configured format instead.
6. IF a filter query is submitted with `date_from` later than `date_to`, THEN THE Attendance_Module SHALL reject the query, return an error indicating the date range is invalid, halt all processing, and perform no search.

---

### Requirement 11: Guard Duty Accomplishment Report — Filing

**User Story:** As a Guard, I want to file a monthly Guard Duty Accomplishment Report with a narrative, so that I can formally document my activities for the reporting period.

#### Acceptance Criteria

1. WHEN an authenticated user with role `guard` or `admin` submits a Guard Duty Accomplishment Report, THE Guard_Report_Module SHALL validate and persist the following fields: `guard_id` (must reference an existing user account with role `guard`), `period_start` (format `YYYY-MM-DD`), `period_end` (format `YYYY-MM-DD`), and `narrative` (non-empty, max 10,000 characters); submissions from users without `guard` or `admin` role SHALL NOT proceed to field validation and SHALL be rejected with an authorization error per Requirement 14.
2. IF a required field (`guard_id`, `period_start`, `period_end`, `narrative`) is absent or empty on submission, THEN THE Guard_Report_Module SHALL reject the request and return a validation error identifying each failing field by name.
3. IF `period_end` is earlier than `period_start` on submission, THEN THE Guard_Report_Module SHALL reject the request and return an error stating that the period end date must not precede the period start date.
4. IF a Guard Duty Accomplishment Report is submitted for a `guard_id` and period that overlaps with an existing non-voided report for the same guard, THEN THE Guard_Report_Module SHALL reject the submission and return an error indicating a conflicting report exists.
5. WHEN a conflicting report is detected, THE Guard_Report_Module SHALL include the existing report's ID in the error response for reference.
6. WHEN a Guard Duty Accomplishment Report is successfully created, THE Audit_Logger SHALL record the creation event, the acting user's ID, and the UTC timestamp.

---

### Requirement 12: Guard Duty Accomplishment Report — Editing and Void

**User Story:** As a Guard or Admin, I want to edit or void a Guard Duty Accomplishment Report, so that corrections can be made and erroneous reports removed from active view without destroying history.

#### Acceptance Criteria

1. WHEN an authenticated user with role `guard` submits an edit to a non-voided Guard Duty Accomplishment Report where `guard_id` matches the acting user's ID, THE Guard_Report_Module SHALL persist the updated `narrative` (non-empty, max 10,000 characters), `period_start`, and `period_end` values, subject to the constraint that `period_start` is not later than `period_end`.
2. WHEN the edit is persisted, THE Guard_Report_Module SHALL instruct THE Audit_Logger to record the before-state, after-state, acting user's ID, and UTC timestamp.
3. IF a user with role `guard` attempts to edit a Guard Duty Accomplishment Report where `guard_id` does not match the acting user's ID, THEN THE Guard_Report_Module SHALL reject the edit and return an authorization error.
4. IF an edit or void request is submitted for a Guard Duty Accomplishment Report that is already voided, THEN THE Guard_Report_Module SHALL reject the request and return an error stating that voided reports cannot be modified.
5. WHEN an authenticated user with role `admin` submits a void request for a non-voided Guard Duty Accomplishment Report with a `void_reason` of 1–500 characters, THE Guard_Report_Module SHALL mark the report as voided and persist the `void_reason`.
6. IF a void request is submitted without a `void_reason`, or with a `void_reason` that is empty or exceeds 500 characters, THEN THE Guard_Report_Module SHALL reject the request and return an error describing the `void_reason` requirement, leaving the report unchanged.
7. THE Guard_Report_Module SHALL NOT permanently delete any Guard Duty Accomplishment Report record.

---

### Requirement 13: Audit Trail

**User Story:** As an Admin, I want every create, edit, and void operation attributed to a user with a timestamp and the full before/after record state, so that I can audit any change to any record in the module.

#### Acceptance Criteria

1. THE Audit_Logger SHALL record an audit entry for every create, edit, and void operation across Pass Slip, Attendance Logbook, and Guard Duty Accomplishment Report records.
2. WHEN THE Audit_Logger records an audit entry, THE Audit_Logger SHALL persist: the table name, the record ID, the operation type (`create`, `edit`, `void`), the before-state as a serialized snapshot (null for creates), the after-state as a serialized snapshot, the acting user's ID, and the UTC timestamp.
3. THE Audit_Logger SHALL NOT overwrite or delete any prior audit entry; the audit log is append-only.
4. WHEN an Admin queries the audit trail for a specific record, THE Audit_Logger SHALL return all audit entries for that record ordered by UTC timestamp ascending.
5. IF the acting user's session is not authenticated at the time of any write operation, THEN THE Pass_Slip_Module, THE Attendance_Module, and THE Guard_Report_Module SHALL each return an authentication error to the caller and SHALL NOT process the request or create an audit entry.
6. IF THE Audit_Logger fails to persist an audit entry for a write operation, THEN THE originating module SHALL roll back the write operation and return an error indication to the caller, ensuring no record state change occurs without a corresponding audit entry.

---

### Requirement 14: Role-Based Access Control

**User Story:** As an Admin, I want role-based access control enforced for all module operations, so that guards, employees, supervisors, and admins can only perform the actions appropriate to their role.

#### Acceptance Criteria

1. WHEN a user with role `employee`, `guard`, or `admin` submits a new Pass Slip Outbound_Leg, THE Pass_Slip_Module SHALL process the request; IF the submitting user's role is not one of these three, THEN THE Pass_Slip_Module SHALL reject the request and return an authorization error indicating insufficient permissions.
2. WHEN a user with role `guard` or `admin` submits a Return_Leg entry, THE Pass_Slip_Module SHALL process the request; IF the submitting user's role is not `guard` or `admin`, THEN THE Pass_Slip_Module SHALL reject the request and return an authorization error.
3. IF a user with role other than `admin` submits a void request for a Pass Slip, THEN THE Pass_Slip_Module SHALL reject the request and return an authorization error.
4. WHEN a user with role `guard` or `admin` submits an attendance log create or edit, THE Attendance_Module SHALL process the request; IF the submitting user's role is not `guard` or `admin`, THEN THE Attendance_Module SHALL reject the request and return an authorization error.
5. IF a user with role other than `admin` submits a void request for an attendance log entry, THEN THE Attendance_Module SHALL first verify the user's authenticated session (per AC 7 of Requirement 14), then reject the request and return an authorization error indicating insufficient role permissions.
6. WHEN a user with role `guard` or `admin` submits a Guard Duty Accomplishment Report creation request, THE Guard_Report_Module SHALL process the request; IF the submitting user's role is not `guard` or `admin`, THEN THE Guard_Report_Module SHALL reject the request and return an authorization error.
7. IF any request arrives without a valid authenticated session, THEN THE Pass_Slip_Module, THE Attendance_Module, and THE Guard_Report_Module SHALL each return an authentication error and SHALL NOT process the request.
8. IF a user with role `employee` requests Pass Slip records where `employee_id` does not match their own user account, THEN THE Pass_Slip_Module SHALL return an authorization error indicating insufficient permissions; users with role `guard`, `supervisor`, or `admin` may view all records.

---

### Requirement 15: Employee Master List Management

**User Story:** As an Admin, I want to maintain the employee master list in the system, so that guards and employees can reference accurate employee names and positions when filing records.

#### Acceptance Criteria

1. WHEN an authenticated user with role `admin` creates an employee record with valid `name` (1–100 characters), `position` (1–100 characters), and `division` (1–100 characters), THE Pass_Slip_Module SHALL persist those fields and assign a system-generated unique `employee_id`; IF any of these fields is absent, empty, or exceeds 100 characters, THEN THE Pass_Slip_Module SHALL reject the submission and return a field-level error message without partially persisting the record.
2. WHEN an authenticated user with role `admin` updates an employee record, THE Audit_Logger SHALL record the prior and new field values, the acting user's ID, and the UTC timestamp.
3. IF an operation is received that would permanently delete an employee record, THEN THE Pass_Slip_Module SHALL reject the operation and return an error indicating permanent deletion is not permitted; void/soft-delete with a reason is required.
4. IF an admin submits a void request for an employee record without a `void_reason`, or with a `void_reason` that is empty or exceeds 500 characters, THEN THE Pass_Slip_Module SHALL reject the request and return an error describing the `void_reason` requirement, leaving the employee record unchanged.
5. WHEN an employee record is voided, THE Pass_Slip_Module SHALL retain all historical Pass Slip and Attendance Logbook records referencing that employee and SHALL display the employee's name with a "(Inactive)" label wherever the employee appears in search results.
6. THE Pass_Slip_Module SHALL NOT synchronize the employee master list with any external HR information system; the list is maintained exclusively through admin data entry.

---

### Requirement 16: Data Integrity — No Hard Deletes

**User Story:** As an Admin, I want the system to prevent permanent deletion of any record, so that the official government records are preserved in accordance with records retention obligations.

#### Acceptance Criteria

1. IF an operation is received that would permanently remove a record from the database in THE Pass_Slip_Module, THE Attendance_Module, or THE Guard_Report_Module, THEN the receiving module SHALL reject the operation and return an error response indicating permanent deletion is not permitted, leaving the record unchanged.
2. WHEN a record is voided, THE Pass_Slip_Module, THE Attendance_Module, and THE Guard_Report_Module SHALL each preserve all field values on the record in an immutable state, including the `void_reason` (1–500 characters, mandatory) and the acting user's ID.
3. IF a void request is submitted without a `void_reason`, or with a `void_reason` that is empty or exceeds 500 characters, THEN the receiving module SHALL reject the request and return an error describing the requirement, leaving the record unchanged.
4. THE Audit_Logger SHALL preserve all audit entries regardless of the voided status of the referenced record.

---

### Requirement 18: Guard Station Kiosk Authentication and Attribution

**User Story:** As a guard on duty, I want to log in to the guard station with a shared kiosk credential and then select my own name before recording any action, so that the system session stays simple while every entry is correctly attributed to the specific guard who performed it.

#### Acceptance Criteria

1. WHEN the guard station workstation authenticates using the shared kiosk credential, THE Pass_Slip_Module and THE Attendance_Module SHALL establish a valid session with role `guard` scoped to that workstation.
2. WHEN an authenticated kiosk session submits a Pass Slip outbound entry, a Return_Leg entry, or an Attendance Logbook entry, THE Pass_Slip_Module or THE Attendance_Module SHALL require the operator to supply a `guard_id` identifying the specific named guard performing the action, selected from the list of active users with role `guard`.
3. IF a kiosk session submits an action without a `guard_id`, THEN THE Pass_Slip_Module or THE Attendance_Module SHALL reject the submission and return a validation error requiring a named guard to be selected before the action can be accepted.
4. IF the `guard_id` supplied by a kiosk session does not reference an active user account with role `guard`, THEN THE Pass_Slip_Module or THE Attendance_Module SHALL reject the submission and return a validation error identifying the unrecognized `guard_id`.
5. WHEN a kiosk session action is successfully persisted, THE Audit_Logger SHALL record the selected `guard_id` (the named guard) as the attribution identity for that action, NOT the shared kiosk account's user ID.
6. THE Pass_Slip_Module and THE Attendance_Module SHALL NOT allow the shared kiosk account's user ID to appear as the `outbound_guard_id`, `return_guard_id`, or `recorded_by` value on any record; only individually named guard IDs are valid for these attribution fields.

---


**User Story:** As an Admin, I want to be aware that historical attendance and pass slip data from the 2020–2026 Excel workbooks will eventually need to be imported, so that this is planned for in a separate spec.

#### Acceptance Criteria

1. WHEN a Pass Slip record is inserted with a `date` value between 2020-01-01 and the current date (inclusive), and `time_out` and `time_in` values in `HH:MM` 24-hour format, THE Pass_Slip_Module SHALL accept the insertion without constraint violations, enabling future bulk load of historical records.
2. WHEN an attendance log entry is inserted with a `date` value between 2020-01-01 and the current date (inclusive), THE Attendance_Module SHALL accept the insertion without constraint violations, enabling future bulk load of historical records.
3. THE Pass_Slip_Module and THE Attendance_Module SHALL NOT expose any UI screen or API endpoint for bulk or batch import of records; any such import capability is deferred to a separate Data Migration spec.
