# Module 04 — Doctor Management

## Purpose
Doctor profiles, credential verification and approval, availability schedules and consultation slots.

## Entities
| Table | Key columns |
|---|---|
| doctors | user_id, specialization, qualification, license_no (unique), registration_council, consultation_fee, approval_status |
| doctor_documents | doctor_id, doc_type, file_url, verified_by, verified_at, verified |
| doctor_schedules | doctor_id, weekday, start_time, end_time, slot_minutes, room_id, facility_id |
| slots | doctor_id, start_at, end_at, status(`OPEN/HELD/BOOKED/BLOCKED`) |
| doctor_leaves | doctor_id, from_at, to_at, reason, approved_by |

`approval_status`: `PENDING → APPROVED/REJECTED`, `APPROVED → ACTIVE`, `ACTIVE ↔ INACTIVE`, `ACTIVE ↔ SUSPENDED`, `REJECTED → PENDING` (resubmit).

## Workflows
1. **Registration & approval** — doctor self-registers → uploads licence and qualification documents → admin verifies each document → approve/reject with reason → on approval the account is activated and the doctor role is granted.
2. **Schedule publishing** — doctor defines weekly templates; the system materialises `slots` for a rolling horizon (e.g. 60 days).
3. **Leave** — approved leave blocks the affected slots and triggers rescheduling notifications to booked patients.
4. **Suspension** — admin suspends; future slots are blocked and appointments flagged for reassignment.

## Role access
| Action | Super Admin | Hospital Admin | Doctor | Receptionist | Patient | Auditor |
|---|---|---|---|---|---|---|
| Doctor profile | Full | Full | Update (own) | Read | Read (public fields) | Read |
| Documents | Full | Full + Verify | Upload (own) | – | – | Read |
| Approval / suspension | Full | Approve | Read (own) | – | – | Read |
| Schedules & slots | Full | Create/Update | Create/Update (own) | Create/Update | Read | Read |

## Rules
- A doctor cannot take appointments unless `approval_status = ACTIVE`.
- `license_no` is unique system-wide; expiry date is tracked and warned on 60 days ahead.
- Slot overlap is rejected; overbooking requires an explicit per-doctor allowance.
- Rejection always carries a reason visible to the doctor.

## Dependencies
Upstream: IAM, Staff, Organisation. Downstream: Appointments, Consultation, OT.
