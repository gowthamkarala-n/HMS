# Module 11 — Operation Theatre (OT)

## Purpose
Surgery scheduling, theatre utilisation, surgical team assignment, consumable usage and operative notes.

## Entities
| Table | Key columns |
|---|---|
| ot_rooms | facility_id, name, status(`AVAILABLE/IN_USE/CLEANING/MAINTENANCE`) |
| ot_bookings | visit_id, ot_room_id, surgery_service_id, surgeon_id, anesthetist_id, scheduled_start, scheduled_end, actual_start, actual_end, status |
| ot_team_members | booking_id, staff_id, role(`SURGEON/ASSISTANT/ANESTHETIST/SCRUB_NURSE/TECHNICIAN`) |
| ot_consumables | booking_id, item_id, batch_id, qty, charged |
| surgery_notes | booking_id, procedure, findings, technique, complications, blood_loss, implants, signed_by, signed_at |
| ot_checklists | booking_id, phase(`SIGN_IN/TIME_OUT/SIGN_OUT`), completed_by, completed_at |

`ot_bookings.status`: `REQUESTED → SCHEDULED → IN_PROGRESS → COMPLETED`, plus `POSTPONED` and `CANCELLED`.

## Workflows
1. **Request** — surgeon raises a booking against an admitted or day-care visit; pre-anaesthetic assessment and consent are prerequisites.
2. **Schedule** — OT coordinator allocates room and time, checks surgeon/anesthetist availability and theatre turnaround.
3. **Pre-op checklist** — WHO-style sign-in / time-out / sign-out, each with a recorded signer.
4. **Perform** — consumables and implants issued from inventory against the booking; stock is deducted through the stock ledger.
5. **Close** — operative note signed; theatre charges, consumable charges and surgeon fees post to the visit; room set to `CLEANING`, housekeeping task raised.

## Role access
| Action | Surgeon (Doctor) | Anesthetist | Nurse | OT Coordinator (Admin) | Inventory | Patient | Auditor |
|---|---|---|---|---|---|---|---|
| Booking | Create (own) | Read | Read (assigned) | Full | – | Read (own) | Read |
| Schedule / reschedule | Request | Read | – | Approve | – | – | Read |
| Checklists | Sign | Sign | Sign | Read | – | – | Read |
| Consumables | Read | Read | Record (assigned) | Read | Read/Issue | – | Read |
| Operative notes | Create/Sign (own) | Add anaesthesia record | Read (assigned) | Metadata only | – | Read (own, post-sign) | Read |

## Rules
- No booking proceeds without consent, pre-anaesthetic clearance and a completed sign-in checklist.
- Implants require batch traceability to the patient.
- Cancellation within the theatre window requires a reason and is reported in utilisation KPIs.

## Dependencies
Upstream: Inpatient, EMR, Doctor, Inventory. Downstream: Billing, Housekeeping.
