# Module 12 — Inpatient & Bed Management

## Purpose
Wards, rooms and beds; admission, bed allocation and transfer; nursing care; housekeeping; discharge. This module is absent from the current repo specification and is required for the hospital to function as a whole entity.

## Entities
| Table | Key columns |
|---|---|
| wards | facility_id, name, type(`GENERAL/ICU/HDU/MATERNITY/ISOLATION/PEDIATRIC`), floor |
| room_types | name, daily_rate_service_id, occupancy |
| rooms | ward_id, room_no, room_type_id |
| beds | room_id, bed_no, status(`VACANT/OCCUPIED/RESERVED/CLEANING/MAINTENANCE/BLOCKED`) |
| admissions | visit_id, patient_id, admitting_doctor_id, admitted_at, expected_discharge, actual_discharge_at, status(`ADMITTED/TRANSFERRED/DISCHARGED/LAMA/EXPIRED`) |
| bed_allocations | admission_id, bed_id, from_at, to_at, daily_rate |
| bed_transfers | admission_id, from_bed_id, to_bed_id, reason, approved_by, at |
| nursing_rounds | admission_id, nurse_id, round_at, observations, vitals_id |
| medication_administration | admission_id, prescription_item_id, given_by, given_at, skipped_reason |
| housekeeping_tasks | bed_id/room_id, type(`CLEANING/LINEN/DISINFECTION/MAINTENANCE`), priority, status(`PENDING/IN_PROGRESS/DONE/VERIFIED`), assigned_to, requested_at, completed_at, verified_by |
| discharge_summaries | admission_id, diagnosis, hospital_course, discharge_meds, follow_up, prepared_by, approved_by |

## Workflows
### Admission
Doctor raises an admission request → bed searched by ward/room type → bed `RESERVED` → admission created and deposit collected → bed `OCCUPIED`.

### Stay
A daily job posts one bed charge per calendar day from `bed_allocations`. Nursing rounds, medication administration, diagnostics, OT and pharmacy issues all post charges to the same `visit_id`, enabling interim bills at any time.

### Transfer
Requested with a reason (e.g. ward → ICU) → approved → previous allocation closed with `to_at`, new allocation opened → rate changes from the transfer timestamp → old bed goes to `CLEANING`.

### Discharge
Discharge initiated → pharmacy returns settled and pending orders closed → discharge summary prepared and approved → final invoice consolidates all interim charges → payment or claim settled → bed set `CLEANING` → housekeeping task auto-created → verified → bed `VACANT`.

### Housekeeping
Trigger (discharge, transfer, soiling report, scheduled disinfection) → task created with type and priority → assigned → `IN_PROGRESS` → `DONE` → nurse verifies → bed released. Each ward type has an SLA; ICU beds cannot be re-allocated before verification.

## Role access
| Action | Hospital Admin | Doctor | Nurse | Housekeeping | Reception | Cashier | Patient | Auditor |
|---|---|---|---|---|---|---|---|---|
| Wards / rooms / beds master | Full | Read | Read | Read | Read | – | – | Read |
| Admission | Full | Create (assigned) | Create (assigned) | – | Create | Read | Read (own) | Read |
| Bed allocation | Full | Request (assigned) | Full (department) | – | Create | – | – | Read |
| Bed transfer | Approve | Request | Create (department) | – | – | – | – | Read |
| Nursing rounds / med admin | Read | Read (assigned) | Full (assigned) | – | – | – | Read (own) | Read |
| Housekeeping tasks | Read | – | Create + Verify (department) | Full (assigned) | – | – | – | Read |
| Discharge summary | Read | Create/Sign (assigned) | Prepare draft | – | – | – | Read (own) | Read |
| Final discharge clearance | Approve | Clinical clearance | – | – | – | Financial clearance | – | Read |

## Rules
- A bed can have only one open `bed_allocations` row.
- Discharge requires both clinical clearance (doctor) and financial clearance (cashier/insurance), except LAMA and medico-legal exceptions.
- Bed status transitions are strictly `OCCUPIED → CLEANING → VACANT`; a bed can never jump to `VACANT` directly.
- Bed-day charging rule (calendar day vs 24-hour block) is a configurable facility policy.

## Dependencies
Upstream: Patient, EMR, Appointment/visits. Downstream: Billing, Insurance, Pharmacy, OT.
