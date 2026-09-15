# Module 08 — Consultation & EMR

## Purpose
The clinical record: encounters, vitals, allergies, coded diagnoses, notes, prescriptions, referrals and follow-ups.

## Entities
| Table | Key columns |
|---|---|
| consultations | visit_id, appointment_id, doctor_id, patient_id, started_at, ended_at, chief_complaint, examination, advice, status(`DRAFT/FINALIZED/AMENDED`) |
| vitals | visit_id, recorded_by, temp, pulse, bp_sys, bp_dia, spo2, resp_rate, height, weight, bmi, recorded_at |
| allergies | patient_id, substance, reaction, severity, recorded_by |
| diagnoses | consultation_id, icd_code, type(`PROVISIONAL/FINAL`), is_primary |
| clinical_notes | consultation_id, body, author_id, signed_at |
| note_amendments | note_id, previous_body, reason, amended_by, amended_at |
| prescriptions | consultation_id, issued_at, status |
| prescription_items | prescription_id, medicine_id, dose, frequency, duration_days, route, quantity, instructions |
| referrals | consultation_id, to_doctor_id / external_name, reason |
| follow_ups | consultation_id, due_date, created_appointment_id |

## Workflows
1. **Start consultation** — doctor opens the checked-in patient; vitals recorded by the nurse are already visible.
2. **Document** — chief complaint, examination, ICD-10 diagnoses (exactly one primary), notes.
3. **Order** — lab tests, imaging, procedures; each order posts to its module queue.
4. **Prescribe** — e-prescription routes to the pharmacy queue; allergy conflicts and duplicate-therapy warnings are raised at prescribing time.
5. **Finalize** — note is signed; a consultation charge is posted. After signing the note is immutable.
6. **Amend** — the authoring doctor may add an amendment with a mandatory reason; the previous body is preserved in `note_amendments`.

## Role access
| Action | Doctor | Nurse | Radiologist | Pharmacist | Lab | Admin | Patient | Auditor |
|---|---|---|---|---|---|---|---|---|
| Consultation record | Full (assigned) | Read (assigned) | Read (assigned) | Read meds only | Read order context | Metadata only | Read (own) | Read |
| Vitals | Read/Update | Create/Update (assigned) | – | – | – | – | Read (own) | Read |
| Allergies | Create/Update | Create | – | Read | – | – | Read (own) | Read |
| Diagnoses & notes | Create/Update (own, pre-sign) | – | – | – | – | – | Read (own) | Read |
| Amend signed note | Approve (own, reason) | – | Approve (own) | – | – | – | – | Read |
| Prescriptions | Full (assigned) | Read | – | Read + dispense | – | Read | Read (own) | Read |

Admins see that an encounter exists, its status and its charges — not the clinical content — unless a break-glass access is logged.

## Rules
- Finalized notes are append-only; corrections are amendments, never overwrites.
- Every diagnosis carries a valid ICD-10 code; free text alone is rejected on finalize.
- Prescribing a scheduled/narcotic drug requires the doctor's licence number on the prescription.
- Access outside the `assigned` scope writes an `access_logs` row.

## Dependencies
Upstream: Appointment/visits, Patient, Doctor, Master data. Downstream: Laboratory, Radiology, Pharmacy, Billing, Inpatient.
