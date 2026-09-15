# Module 06 — Appointment & Queue Management

## Purpose
OPD booking, rescheduling, check-in, the live waiting queue, and the `visits` episode that ties every later charge together.

## Entities
| Table | Key columns |
|---|---|
| visits | patient_id, visit_no, type(`OPD/IPD/ER/DAYCARE`), facility_id, opened_at, closed_at, status |
| appointments | visit_id, patient_id, doctor_id, slot_id, scheduled_at, type(`NEW/FOLLOWUP/TELE`), status, cancel_reason, booked_by |
| queue_tokens | visit_id, department_id, token_no, issued_at, called_at, served_by |
| appointment_reminders | appointment_id, channel, scheduled_at, sent_at |

`appointments.status`: `SCHEDULED → CONFIRMED → CHECKED_IN → IN_CONSULT → COMPLETED`, plus `CANCELLED` and `NO_SHOW`.

## Workflows
1. **Booking** — patient or reception picks a doctor and an `OPEN` slot → slot moves to `HELD` for 10 minutes → confirmation moves it to `BOOKED` and opens a `visit` on the appointment date.
2. **Check-in** — reception verifies identity → appointment `CHECKED_IN` → token issued → patient appears in the doctor's live queue.
3. **Reschedule / cancel** — old slot released, policy-based cancellation window; late cancellation can carry a fee.
4. **No-show** — auto-marked N minutes after the slot ends; repeat no-shows are flagged at booking.

## Role access
| Action | Hospital Admin | Receptionist | Doctor | Nurse | Patient | Auditor |
|---|---|---|---|---|---|---|
| Book / reschedule | Full | Full | Own patients | – | Own | Read |
| Cancel | Full | Full | Own | – | Own (within window) | Read |
| Check-in / tokens | Full | Full | Read (own queue) | Update (department) | Read (own) | Read |
| Queue display | Full | Full | Read (own) | Read (department) | Read (own) | Read |

## Rules
- One open `visit` per patient per facility per day for OPD; IPD and ER open their own visit.
- Slots cannot be double-booked unless the doctor's overbooking allowance is set.
- Booking is blocked for doctors who are not `ACTIVE`.
- Every charge, order and invoice references a `visit_id`.

## Dependencies
Upstream: Patient, Doctor, Organisation. Downstream: Consultation, Billing, Inpatient.
