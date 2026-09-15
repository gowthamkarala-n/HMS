
# HMS Module Reference — Index

One reference document per module. Each file follows the same structure: purpose, entities, workflows, role access, rules, dependencies.

Companion document: [`HMS-Design-Review.md`](../HMS-Design-Review.md) — gap analysis against the original spec, consolidated entity model, full role/permission/scope matrix, revised workflows and the file-by-file change list.

| # | Module | File |
|---|---|---|
| 01 | Identity & Access Management | [01-identity-access-management.md](01-identity-access-management.md) |
| 02 | Organisation & Master Data | [02-organisation-master-data.md](02-organisation-master-data.md) |
| 03 | Staff Management | [03-staff-management.md](03-staff-management.md) |
| 04 | Doctor Management | [04-doctor-management.md](04-doctor-management.md) |
| 05 | Patient Management | [05-patient-management.md](05-patient-management.md) |
| 06 | Appointment & Queue | [06-appointment-queue.md](06-appointment-queue.md) |
| 07 | Emergency & Triage | [07-emergency-triage.md](07-emergency-triage.md) |
| 08 | Consultation & EMR | [08-consultation-emr.md](08-consultation-emr.md) |
| 09 | Laboratory & Diagnostics | [09-laboratory.md](09-laboratory.md) |
| 10 | Radiology & Imaging | [10-radiology-imaging.md](10-radiology-imaging.md) |
| 11 | Operation Theatre | [11-operation-theatre.md](11-operation-theatre.md) |
| 12 | Inpatient & Bed Management | [12-inpatient-bed-management.md](12-inpatient-bed-management.md) |
| 13 | Pharmacy & Dispensing | [13-pharmacy-dispensing.md](13-pharmacy-dispensing.md) |
| 14 | Inventory & Supply Chain | [14-inventory-supply-chain.md](14-inventory-supply-chain.md) |
| 15 | Billing & Financials | [15-billing-financials.md](15-billing-financials.md) |
| 16 | Insurance & Claims | [16-insurance-claims.md](16-insurance-claims.md) |
| 17 | Notification & Communication | [17-notifications.md](17-notifications.md) |
| 18 | Reporting, Audit & Security | [18-reporting-audit-security.md](18-reporting-audit-security.md) |

## Cross-cutting conventions
- UUID primary keys; `created_at`, `updated_at`, `created_by`, `updated_by` on every table.
- `facility_id` on every operational and clinical table (multi-branch ready).
- Soft delete on master data; clinical records are append-only with amendment rows.
- Money as integer minor units plus currency code.
- Access is `role → permission (module.entity.action) → scope (own/assigned/department/facility/global)`.
- Every charge, order and clinical event references a `visit_id`.
