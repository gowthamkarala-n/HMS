# Module 07 — Emergency & Triage

## Purpose
Emergency intake, acuity scoring and the triage queue that decides whether a patient goes to OPD, IPD, OT or discharge.

## Entities
| Table | Key columns |
|---|---|
| er_cases | visit_id, arrival_mode(`WALK_IN/AMBULANCE/REFERRED/POLICE`), chief_complaint, arrival_at, mlc_flag |
| triage_assessments | visit_id, er_case_id, acuity(1–5), vitals_id, assessed_by, assessed_at, disposition(`OPD/IPD/OT/OBSERVATION/DISCHARGE/REFER_OUT`) |
| mlc_records | er_case_id, police_station, informed_at, officer_name |

Acuity 1 = resuscitation, 5 = non-urgent. Queue order is acuity first, arrival time second.

## Workflows
1. **Arrival** — ER visit opened without prior appointment; registration may be provisional (unknown patient gets a temporary MRN, merged later).
2. **Triage** — nurse records vitals and assigns acuity → patient enters the acuity-ordered queue.
3. **Disposition** — ER doctor treats and dispositions: admit (creates an `admissions` row), send to OT, observe, discharge, or refer out.
4. **Medico-legal case** — MLC flag mandates a police intimation record and locks the file from ordinary edits.

## Role access
| Action | Hospital Admin | Nurse | ER Doctor | Receptionist | Patient | Auditor |
|---|---|---|---|---|---|---|
| Open ER case | Read | Create | Create | Create | – | Read |
| Triage assessment | Read | Full (department) | Create/Update (assigned) | – | – | Read |
| Disposition | Read | – | Create (assigned) | – | – | Read |
| MLC record | Read | Create | Create | – | – | Read |

## Rules
- Treatment is never blocked by payment in ER; billing follows care.
- Provisional registration must be reconciled to a real MRN before discharge.
- Acuity 1 and 2 cases raise an immediate notification to the on-duty doctor.
- MLC records are append-only.

## Dependencies
Upstream: Patient, Appointment (visits). Downstream: EMR, Inpatient, OT, Billing.
