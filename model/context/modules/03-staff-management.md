# Module 03 — Staff Management

## Purpose
Employment records, department posting, shift rosters and attendance for all non-doctor staff (and the employment side of doctors).

## Entities
| Table | Key columns |
|---|---|
| staff | user_id, employee_no, department_id, designation, join_date, exit_date, status(`ACTIVE/ON_LEAVE/EXITED`) |
| staff_documents | staff_id, doc_type, file_url, verified_by, verified_at |
| staff_shifts | staff_id, shift_date, start_at, end_at, ward_id, shift_type(`MORNING/EVENING/NIGHT`) |
| attendance | staff_id, date, check_in, check_out, status(`PRESENT/ABSENT/LEAVE/HALF_DAY`) |
| leave_requests | staff_id, type, from_date, to_date, status(`REQUESTED/APPROVED/REJECTED`), approved_by |

## Workflows
1. **Onboarding** — HR creates staff record → IAM user created → role assigned with facility/department scope → documents verified.
2. **Roster** — department head publishes weekly shifts; nurse assignment to wards drives the `assigned` scope used by EMR and IPD modules.
3. **Leave** — request → department head approval → roster auto-adjusted.
4. **Exit** — exit date set → roles expire → sessions revoked.

## Role access
| Action | Super Admin | Hospital Admin | Dept head / Lab Mgr | Staff (own) | Auditor |
|---|---|---|---|---|---|
| Staff records | Full | Full | Read (department) | Read (own) | Read |
| Shifts | Full | Full | Create/Update (department) | Read (own) | Read |
| Attendance | Full | Full | Update (department) | Read (own) | Read |
| Leave approval | Full | Approve | Approve (department) | Request | Read |

## Rules
- "Assigned" scope elsewhere in the system is computed from today's `staff_shifts` rows.
- Exiting a staff member never deletes clinical records they authored.
- Salary/payroll is explicitly out of scope for this HMS.

## Dependencies
Upstream: IAM, Organisation. Downstream: Inpatient (nurse assignment), Laboratory, OT.
