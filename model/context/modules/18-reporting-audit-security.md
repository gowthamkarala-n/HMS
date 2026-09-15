# Module 18 — Reporting, Audit & Security

## Purpose
Operational dashboards and KPIs, the immutable audit trail, medical-record access tracking and the platform security posture.

## Entities
| Table | Key columns |
|---|---|
| audit_logs | user_id, action, entity, entity_id, old_value, new_value, ip, at |
| access_logs | user_id, patient_id, purpose, at, break_glass |
| report_definitions | code, name, query_ref, allowed_roles, default_scope |
| report_runs | definition_id, run_by, params, row_count, at |

## Dashboards
- **Hospital Admin** — patients today, active doctors, appointments, occupancy %, revenue today/month, pending doctor approvals, low-stock and expiry alerts, claim ageing.
- **Doctor** — today's appointments, live queue, pending lab/imaging reports, admitted patients, prescriptions issued.
- **Nurse** — assigned beds, due medications, pending vitals, housekeeping status.
- **Pharmacist** — dispensing queue, near-expiry batches, stock-outs, counter sales.
- **Lab / Radiology** — pending collections, TAT breaches, unverified results.
- **Inventory** — reorder list, open POs, GRN pending, write-offs.
- **Cashier / Insurance** — collections by method, outstanding invoices, claims by status and ageing.
- **Patient** — upcoming appointments, active prescriptions, reports, pending payments.

## Audit
Every create, update, approve, void and export writes an `audit_logs` row with before/after values, captured at the service layer (AOP/middleware), asynchronously. Any read of a patient record outside the reader's `assigned` scope writes an `access_logs` row with a mandatory purpose and, where applicable, a break-glass flag that notifies the admin.

## Role access
| Action | Super Admin | Hospital Admin | Auditor | Department roles | Patient |
|---|---|---|---|---|---|
| Dashboards | Global | Facility | Read all | Department scope | Own |
| Report export | Full | Facility | Full | Department (non-PHI) | Own |
| Audit logs | Read | Read | Full | – | – |
| Access logs | Read | Read | Full | – | Own record access history |
| Security settings | Full | Read | Read | – | – |

## Security posture
- Passwords hashed (BCrypt/Argon2); MFA for admin and clinical roles.
- TLS 1.3 in transit; sensitive clinical fields encrypted at rest.
- Access enforced as `permission + scope` at the data layer, not only in the UI.
- Rate limiting and strict CORS on all APIs; session revocation on role change or exit.
- Audit and access logs are append-only, retained per regulatory policy, and exportable for HIPAA/GDPR requests.
- Patient data export and erasure requests are handled as controlled, audited workflows.

## Dependencies
Upstream: every module. Downstream: none.
