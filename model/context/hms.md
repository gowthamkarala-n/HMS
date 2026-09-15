# HMS — Design Review, Module Map, Entity Model & Role Access

Reviewed source: `gowthamkarala-n/HMS` → `model/HMS.md` and the 11 module documents under `model/HMS/`.

This document (1) reviews the current specification, (2) proposes the corrected module map, (3) defines the entity model per module, (4) defines the role/permission/scope model with a full access matrix, (5) revises the core workflows, and (6) lists the concrete edits to make in the repository.

Note on stack: the repo documents a Spring Boot + MySQL implementation. The entity and access model below is written stack-neutral. Where an implementation note is needed for the Lovable build (TanStack Start + Postgres with row-level security), it is marked *Impl*.

---

## 1. Review summary

### Strong in the current spec
- 13 modules with clean bounded contexts and a modular-monolith package layout that can be split later.
- Doctor approval and patient-visit state machines are well formed.
- FEFO dispensing, barcode sample tracking, PO/GRN inventory, consolidated invoicing.
- Audit logging, JWT/RBAC, HIPAA/GDPR intent.

### Gaps
| # | Gap | Impact |
|---|---|---|
| 1 | No inpatient module (wards, rooms, beds, admission, housekeeping) | Visit lifecycle dead-ends at `ADMITTED`; no bed charges |
| 2 | No insurance/TPA entities | Billing cannot split payer vs patient liability |
| 3 | No triage or operation theatre entities | `TRIAGE` state has no data behind it |
| 4 | Radiology folded into Laboratory | Different scheduling, modality and report workflow |
| 5 | Billing has no `invoice_items`, service price master, tax/discount/refund | Cannot produce a real bill |
| 6 | No vitals, allergies, ICD-10 diagnoses, documents, referrals tables | EMR features have no storage |
| 7 | `doctor_schedules`, `slots`, `queue_tokens`, `inventory_transactions`, `purchase_orders`, `grn`, `departments`, `notifications` named but undefined | Workflows unimplementable as written |
| 8 | Flat role list; no scope concept | Cannot express "own patients only" or department scoping |
| 9 | Nurse, Cashier, Radiologist, Housekeeping, Insurance desk, Auditor roles missing | Real staff cannot be modelled |
| 10 | Clinical records implied editable | Compliance requires append-only amendments |
| 11 | No `facility_id` on core tables | Blocks a second branch/hospital |

---

## 2. Module map (18 modules)

| # | Module | Purpose | Owns | Depends on |
|---|---|---|---|---|
| 1 | Identity & Access (IAM) | Auth, roles, permissions, sessions, MFA | users, roles, permissions, user_roles, sessions | — |
| 2 | Organisation & Master Data | Facilities, departments, service & price catalogue, code sets | facilities, departments, services, price_list, icd_codes | IAM |
| 3 | Staff Management | Employment records, shifts, attendance | staff, staff_shifts, attendance | IAM, Org |
| 4 | Doctor Management | Doctor profile, credential approval, availability | doctors, doctor_documents, doctor_schedules, slots | IAM, Staff |
| 5 | Patient Management | Demographics, MRN, contacts, consents | patients, patient_contacts, patient_consents, patient_documents | IAM |
| 6 | Appointment & Queue | OPD booking, rescheduling, check-in, tokens | appointments, queue_tokens, visits | Doctor, Patient |
| 7 | Emergency & Triage | ER intake, acuity scoring, triage queue | er_cases, triage_assessments | Patient, Appointment |
| 8 | Consultation & EMR | Encounters, vitals, diagnoses, notes, prescriptions | consultations, vitals, allergies, diagnoses, clinical_notes, note_amendments, prescriptions, prescription_items, referrals | Appointment, Master Data |
| 9 | Laboratory & Diagnostics | Test catalogue, orders, samples, results | lab_tests, lab_panels, lab_orders, lab_order_items, lab_samples, lab_results | EMR |
| 10 | Radiology & Imaging | Modality scheduling, studies, reports | imaging_services, imaging_orders, imaging_studies, imaging_reports, modalities | EMR |
| 11 | Operation Theatre | OT scheduling, surgical teams, consumables, notes | ot_rooms, ot_bookings, ot_team_members, surgery_notes | IPD, EMR, Inventory |
| 12 | Inpatient & Bed Management | Wards, rooms, beds, admission, transfer, discharge, housekeeping | wards, rooms, beds, admissions, bed_allocations, bed_transfers, nursing_rounds, housekeeping_tasks, discharge_summaries | Patient, EMR, Billing |
| 13 | Pharmacy & Dispensing | Medicine catalogue, batches, FEFO dispensing, POS | medicines, medicine_batches, pharmacy_orders, dispenses, dispense_items, returns | EMR, Inventory, Billing |
| 14 | Inventory & Supply Chain | Items, suppliers, PO, GRN, stock ledger, expiry | items, suppliers, purchase_orders, po_items, grns, grn_items, stock_ledger, stock_transfers, stock_adjustments | Org |
| 15 | Billing & Financials | Charge capture, invoices, payments, refunds, credit notes | charges, invoices, invoice_items, payments, refunds, credit_notes, tax_rates, discounts | all revenue modules |
| 16 | Insurance & Claims | Payers, policies, pre-auth, claims, settlements | payers, patient_policies, preauthorizations, claims, claim_items, settlements | Patient, Billing |
| 17 | Notification & Communication | Templates, events, delivery, preferences | notification_templates, notifications, delivery_logs, user_preferences | all |
| 18 | Reporting, Audit & Security | Dashboards, KPI, audit trail, access logs | audit_logs, access_logs, report_definitions | all |

Removed from the original list as standalone: "Pharmacy Workflow" (merged into 13) and "User & Staff Management" split into IAM (1) + Staff (3).

---

## 3. Entity model

Conventions applied to every table:
- `id` UUID primary key, `created_at`, `updated_at`, `created_by`, `updated_by`.
- `facility_id` on every operational/clinical table (multi-branch tenancy key).
- `deleted_at` soft delete on master data. Clinical records are **never** updated in place — corrections create an amendment row.
- All money as integer minor units + `currency`; all status columns are enums.

*Impl: Postgres, one `user_roles` table separate from profiles, RLS policies driven by a `has_permission(user, permission, scope)` security-definer function.*

### 3.1 IAM
| Table | Key columns |
|---|---|
| users | username, email, phone, password_hash, status(`ACTIVE/INACTIVE/LOCKED`), mfa_enabled, last_login_at |
| roles | code, name, is_system |
| permissions | code (`module.entity.action`), description |
| role_permissions | role_id, permission_id, scope(`own/assigned/department/facility/global`) |
| user_roles | user_id, role_id, facility_id, department_id, valid_from, valid_to |
| sessions | user_id, refresh_token_hash, ip, user_agent, expires_at, revoked_at |

### 3.2 Organisation & master data
facilities(code, name, address, timezone) · departments(facility_id, code, name, head_staff_id) · services(code, name, department_id, category `CONSULT/LAB/IMAGING/PROCEDURE/BED/PHARMACY`, is_active) · price_list(service_id, payer_type, amount, valid_from, valid_to) · tax_rates · icd_codes(code, title, version) · loinc_codes (lab) · uoms.

### 3.3 Staff & doctor
staff(user_id, employee_no, department_id, designation, join_date, status) · staff_shifts(staff_id, shift_date, start_at, end_at, ward_id) · attendance.
doctors(user_id, specialization, qualification, license_no, registration_council, consultation_fee, approval_status `PENDING/APPROVED/REJECTED/ACTIVE/INACTIVE/SUSPENDED`) · doctor_documents(doc_type, file_url, verified_by, verified_at) · doctor_schedules(doctor_id, weekday, start_time, end_time, slot_minutes, room_id) · slots(doctor_id, start_at, end_at, status `OPEN/HELD/BOOKED/BLOCKED`) · doctor_leaves.

### 3.4 Patient
patients(user_id nullable, mrn unique per facility, first_name, last_name, dob, gender, blood_group, phone, address, is_deceased) · patient_contacts(relation, name, phone, is_emergency) · patient_consents(type, granted_at, document_url) · patient_documents · patient_merges(source_patient_id, target_patient_id, merged_by) — duplicate MRN handling, currently missing.

### 3.5 Appointment, queue & visit
appointments(patient_id, doctor_id, slot_id, scheduled_at, type `NEW/FOLLOWUP/TELE`, status `SCHEDULED/CONFIRMED/CHECKED_IN/IN_CONSULT/COMPLETED/CANCELLED/NO_SHOW`, cancel_reason, booked_by) · visits(patient_id, visit_no, type `OPD/IPD/ER/DAYCARE`, opened_at, closed_at, status) — new entity that ties every charge to one episode · queue_tokens(visit_id, department_id, token_no, called_at, served_by).

### 3.6 Emergency & triage
er_cases(visit_id, arrival_mode, chief_complaint, arrival_at) · triage_assessments(er_case_id or visit_id, acuity `1..5`, vitals_id, assessed_by nurse, assessed_at, disposition `OPD/IPD/OT/DISCHARGE`).

### 3.7 Consultation & EMR
consultations(visit_id, appointment_id, doctor_id, patient_id, started_at, ended_at, chief_complaint, examination, advice, status `DRAFT/FINALIZED/AMENDED`) · vitals(visit_id, recorded_by, temp, pulse, bp_sys, bp_dia, spo2, resp_rate, height, weight, bmi, recorded_at) · allergies(patient_id, substance, reaction, severity, recorded_by) · diagnoses(consultation_id, icd_code, type `PROVISIONAL/FINAL`, is_primary) · clinical_notes(consultation_id, body, author_id, signed_at) · note_amendments(note_id, previous_body, reason, amended_by, amended_at) — append-only compliance requirement · prescriptions(consultation_id, issued_at, status) · prescription_items(medicine_id, dose, frequency, duration_days, route, quantity, instructions) · referrals(consultation_id, to_doctor_id/external, reason) · follow_ups(consultation_id, due_date, created_appointment_id).

### 3.8 Laboratory
lab_tests(service_id, loinc_code, specimen_type, normal_range_low/high, unit, tat_minutes) · lab_panels / lab_panel_items · lab_orders(visit_id, consultation_id, ordered_by, priority `ROUTINE/URGENT/STAT`, status `ORDERED/COLLECTED/IN_PROCESS/COMPLETED/VERIFIED/CANCELLED`) · lab_order_items(lab_test_id, status) · lab_samples(lab_order_id, barcode, collected_by, collected_at, rejected_reason) · lab_results(lab_order_item_id, value, unit, flag `LOW/NORMAL/HIGH/CRITICAL`, entered_by, verified_by, verified_at) · critical_result_alerts.

### 3.9 Radiology
modalities(code `XRAY/CT/MRI/USG`, room_id) · imaging_services(service_id, modality_id, prep_instructions, duration_minutes) · imaging_orders(visit_id, ordered_by, status) · imaging_studies(order_id, modality_id, scheduled_at, performed_by, accession_no, dicom_uid, image_urls) · imaging_reports(study_id, findings, impression, radiologist_id, signed_at, addendum_of).

### 3.10 Operation theatre
ot_rooms(facility_id, name, status) · ot_bookings(visit_id, ot_room_id, surgery_service_id, surgeon_id, anesthetist_id, scheduled_start, scheduled_end, status `REQUESTED/SCHEDULED/IN_PROGRESS/COMPLETED/CANCELLED`) · ot_team_members(booking_id, staff_id, role) · ot_consumables(booking_id, item_id, batch_id, qty) · surgery_notes(booking_id, procedure, findings, complications, signed_by).

### 3.11 Inpatient & bed management
wards(facility_id, name, type `GENERAL/ICU/HDU/MATERNITY/ISOLATION`, floor) · rooms(ward_id, room_no, room_type_id) · room_types(name, daily_rate_service_id) · beds(room_id, bed_no, status `VACANT/OCCUPIED/RESERVED/CLEANING/MAINTENANCE/BLOCKED`) · admissions(visit_id, patient_id, admitting_doctor_id, admitted_at, expected_discharge, actual_discharge_at, status `ADMITTED/TRANSFERRED/DISCHARGED/LAMA/EXPIRED`) · bed_allocations(admission_id, bed_id, from_at, to_at, daily_rate) — drives automatic daily bed charges · bed_transfers(admission_id, from_bed_id, to_bed_id, reason, approved_by) · nursing_rounds(admission_id, nurse_id, round_at, observations, vitals_id) · medication_administration(admission_id, prescription_item_id, given_by, given_at) · housekeeping_tasks(bed_id/room_id, type `CLEANING/LINEN/DISINFECTION/MAINTENANCE`, status `PENDING/IN_PROGRESS/DONE/VERIFIED`, assigned_to, requested_at, completed_at, verified_by) · discharge_summaries(admission_id, diagnosis, course, discharge_meds, follow_up, prepared_by, approved_by).

### 3.12 Pharmacy
medicines(item_id, generic_name, brand, strength, form, schedule `OTC/H/H1/NARCOTIC`, requires_prescription) · medicine_batches(medicine_id, batch_no, expiry_date, mrp, cost, qty_on_hand, location) · pharmacy_orders(prescription_id or walk-in, status `QUEUED/PARTIAL/DISPENSED/CANCELLED`) · dispenses(order_id, dispensed_by, dispensed_at, invoice_id) · dispense_items(medicine_id, batch_id, qty, unit_price, substitution_of) · pharmacy_returns(dispense_item_id, qty, reason, approved_by) · fefo_overrides(dispense_item_id, reason, approved_by) — override must be logged, currently unmodelled.

### 3.13 Inventory
items(code, name, category `DRUG/CONSUMABLE/ASSET/LINEN`, uom, reorder_level, is_batch_tracked) · suppliers · supplier_items(price, lead_time_days) · purchase_orders(supplier_id, status `DRAFT/SENT/PARTIAL/RECEIVED/CANCELLED`, expected_at, approved_by) · po_items · grns(po_id, received_by, invoice_no, received_at) · grn_items(item_id, batch_no, expiry, qty, unit_cost, landed_cost) · stock_ledger(item_id, batch_id, location_id, txn_type `GRN/ISSUE/DISPENSE/RETURN/ADJUST/TRANSFER/EXPIRY_WRITEOFF`, qty_delta, ref_type, ref_id) — single source of truth for stock · stock_transfers · stock_adjustments(reason, approved_by) · expiry_writeoffs.

### 3.14 Billing
charges(visit_id, service_id, source_module, source_ref_id, qty, unit_price, discount, tax, net_amount, status `UNBILLED/BILLED/CANCELLED`) — charge capture layer missing today · invoices(visit_id, patient_id, invoice_no, type `INTERIM/FINAL`, subtotal, discount_total, tax_total, payer_amount, patient_amount, total, status `DRAFT/ISSUED/PARTIALLY_PAID/PAID/VOID`) · invoice_items(charge_id, description, qty, unit_price, amount) · payments(invoice_id, amount, method `CASH/CARD/UPI/BANK/INSURANCE/WALLET`, transaction_ref, received_by, received_at) · refunds(payment_id, amount, reason, approved_by) · credit_notes(invoice_id, amount, reason, approved_by) · deposits(visit_id, amount) — IPD advances · discounts(type, amount/percent, reason, approved_by).

### 3.15 Insurance & claims
payers(name, type `INSURER/TPA/CORPORATE/GOVT`, contact) · payer_contracts(payer_id, price_list_id, discount_terms) · patient_policies(patient_id, payer_id, policy_no, valid_from, valid_to, sum_insured, balance) · preauthorizations(visit_id, policy_id, requested_amount, approved_amount, status `REQUESTED/APPROVED/PARTIAL/REJECTED`, reference_no) · claims(invoice_id, policy_id, claim_no, claimed_amount, status `DRAFT/SUBMITTED/QUERIED/APPROVED/REJECTED/SETTLED`) · claim_items · claim_documents · settlements(claim_id, settled_amount, tds, settled_at, utr).

### 3.16 Notifications, audit
notification_templates(code, channel `EMAIL/SMS/PUSH/INAPP`, subject, body) · notifications(user_id, template_code, payload, status, scheduled_at) · delivery_logs(provider, provider_ref, status, error) · user_notification_preferences.
audit_logs(user_id, action, entity, entity_id, old_value, new_value, ip, at) · access_logs(user_id, patient_id, purpose, at) — required for medical-record access tracking, currently missing.

---

## 4. Role & access model

### 4.1 Model
Access is `role → permission → scope`, not role → module.

- **Permission code**: `module.entity.action`, e.g. `pharmacy.dispense.create`, `emr.note.amend`, `billing.invoice.void`.
- **Actions**: `read`, `create`, `update`, `delete`, `approve`, `export`.
- **Scope**: `own` (records belonging to the user/patient) · `assigned` (patients or beds assigned to the user today) · `department` · `facility` · `global`.

A permission grant is only meaningful with a scope: a Doctor has `emr.consultation.read@assigned`, an Auditor has `emr.consultation.read@facility` but no write permission anywhere.

### 4.2 Roles (15)
Super Admin · Hospital Admin · Doctor · Nurse · Receptionist · Cashier · Pharmacist · Lab Technician · Lab Manager · Radiologist · Inventory Manager · Ward/Housekeeping · Insurance Desk · Auditor · Patient.

### 4.3 Access matrix

Legend: **F** full (CRUD) · **W** create/update · **R** read · **A** approve · **O** own only · **AS** assigned only · **D** department scope · **–** no access. Scope suffix applies to all letters in the cell.

| Module | Super Admin | Hosp. Admin | Doctor | Nurse | Reception | Cashier | Pharmacist | Lab Tech | Lab Mgr | Radiologist | Inventory | Housekeeping | Insurance | Auditor | Patient |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| System config | F | R | – | – | – | – | – | – | – | – | – | – | – | R | – |
| Facilities / departments | F | W | R | R | R | R | R | R | R | R | R | R | R | R | – |
| Service & price master | F | W+A | R | – | R | R | R | R | R | R | R | – | R | R | – |
| Users & roles | F | F | – | – | – | – | – | – | – | – | – | – | – | R | – |
| Staff & shifts | F | F | R (D) | R (D) | R | – | R | R | W (D) | R | R | R (O) | – | R | – |
| Doctor onboarding/approval | F | F+A | R (O) | – | R | – | – | – | – | R (O) | – | – | – | R | – |
| Doctor schedules | F | W | W (O) | R | W | – | – | – | – | W (O) | – | – | – | R | R |
| Patients (demographics) | R | F | R (AS) | R (AS) | F | R | R | R (AS) | R (AS) | R (AS) | – | – | R | R | R (O) |
| Appointments | R | F | W (O) | R (D) | F | R | – | – | – | W (O) | – | – | – | R | W (O) |
| Queue / check-in | R | F | R (O) | W (D) | F | – | R (D) | R (D) | R (D) | R (D) | – | – | – | R | R (O) |
| Emergency & triage | R | R | W (AS) | F (D) | W | – | – | – | – | R | – | – | – | R | R (O) |
| Consultation & EMR | R* | R* | F (AS) | W vitals (AS) | – | – | R meds (AS) | R order (AS) | R order (AS) | R (AS) | – | – | R summary | R | R (O) |
| EMR amendment | – | – | A (O, reason) | – | – | – | – | – | – | A (O) | – | – | – | R | – |
| Prescriptions | R | R | F (AS) | R (AS) | – | – | R + dispense | – | – | – | – | – | – | R | R (O) |
| Laboratory | R | R | W order (AS), R result | R (AS) | R status | – | – | F sample+result entry | F + verify | – | – | – | R | R | R (O) |
| Radiology | R | R | W order (AS), R report | R (AS) | W schedule | – | – | – | – | F | – | – | R | R | R (O) |
| Operation theatre | R | W | W (O) | W (AS) | R | – | – | – | – | – | R consumables | R | – | R | R (O) |
| Inpatient: admission/discharge | R | F | W (AS) | W (AS) | W admit | – | – | – | – | – | – | – | R | R | R (O) |
| Beds & wards (master) | F | F | R | R | R | – | – | – | – | – | – | R | – | R | – |
| Bed allocation / transfer | R | F | W (AS) | F (D) | W | – | – | – | – | – | – | – | – | R | – |
| Housekeeping tasks | R | R | – | W (D) | – | – | – | – | – | – | – | F (AS) | – | R | – |
| Nursing rounds / med admin | R | R | R (AS) | F (AS) | – | – | – | – | – | – | – | – | – | R | R (O) |
| Pharmacy dispensing | R | R | R | R | – | R | F | – | – | – | R | – | – | R | R (O) |
| FEFO override | – | A | – | – | – | – | W (reason) | – | – | – | – | – | – | R | – |
| Medicine & batch master | R | W | R | – | – | – | F | – | – | – | F | – | – | R | – |
| Inventory: items, suppliers | R | W | – | – | – | – | R | R | R | R | F | R | – | R | – |
| Purchase orders | R | A | – | – | – | – | W | – | W | – | F | – | – | R | – |
| GRN / stock ledger | R | R | – | – | – | – | W | W | W | – | F | W linen | – | R | – |
| Stock adjustment / write-off | R | A | – | – | – | – | W | – | W | – | W | – | – | R | – |
| Charges (capture) | R | R | auto | auto | W | W | auto | auto | auto | auto | – | auto | – | R | R (O) |
| Invoices | R | F | R (O) | – | W | F | W pharmacy | – | – | – | – | – | R | R | R (O) |
| Payments & receipts | R | R | – | – | W | F | W | – | – | – | – | – | W | R | W (O) |
| Discounts / refunds / void | R | A | – | – | – | W (request) | – | – | – | – | – | – | – | R | – |
| Insurance: policies & pre-auth | R | R | R (AS) | – | W | R | – | – | – | – | – | – | F | R | R (O) |
| Claims & settlements | R | R | – | – | – | R | – | – | – | – | – | – | F | R | R (O) |
| Notifications (templates) | F | W | – | – | – | – | – | – | – | – | – | – | – | R | – |
| Reports & analytics | F | F (facility) | R (O) | R (D) | R (D) | R (D) | R (D) | R (D) | R (D) | R (D) | R (D) | R (AS) | R (D) | R | – |
| Audit & access logs | R | R | – | – | – | – | – | – | – | – | – | – | – | F | – |

`R*` = Super Admin / Hospital Admin see EMR metadata (that an encounter exists, its charges and status), not clinical content, unless a break-glass access is recorded in `access_logs`. This replaces the current spec's blanket "Admin: Read" on Consultations.

### 4.4 Sensitive-action rules
| Action | Who | Control |
|---|---|---|
| Amend a finalized clinical note | Authoring doctor only | Original preserved in `note_amendments`, reason mandatory |
| View another doctor's patient record | Any doctor, break-glass | Written to `access_logs`, reason mandatory, admin notified |
| Void / cancel an issued invoice | Hospital Admin | Credit note required, reason logged |
| Refund a payment | Cashier requests → Hospital Admin approves | Dual control |
| FEFO override at dispensing | Pharmacist writes, Hospital Admin approves | `fefo_overrides` row, reason mandatory |
| Stock write-off above threshold | Inventory Manager requests → Admin approves | Dual control |
| Reopen a discharged file | Hospital Admin | Time-limited, audited |
| Change price list | Hospital Admin + effective-dated | No retroactive edits; new row with `valid_from` |
| Delete a patient record | Nobody | Soft-deactivate + merge only |

---

## 5. Revised workflows

### 5.1 Unified visit lifecycle (OPD + ER + IPD)
```text
                    ┌──────────► NO_SHOW
SCHEDULED ─► CHECKED_IN ─► TRIAGE ─► READY_FOR_DOCTOR ─► IN_CONSULTATION
                                                             │
                     ┌───────────────────────────────────────┤
                     ▼                                       ▼
            PENDING_DIAGNOSTICS ──► (results) ──► IN_CONSULTATION
                                                             │
                     ┌───────────────────────────────────────┤
                     ▼                                       ▼
                 ADMITTED                              PENDING_BILLING
                     │                                       │
   BED_ALLOCATED ─► UNDER_TREATMENT ─► (OT / transfers)      ▼
                     │                                  DISCHARGED
                     ▼
        DISCHARGE_INITIATED ─► FINAL_BILLING ─► DISCHARGED
                     └─► LAMA / EXPIRED / TRANSFERRED_OUT
```

### 5.2 Inpatient flow
Admission request (doctor) → bed search by ward/type → reserve bed (`RESERVED`) → admission created, deposit collected → bed `OCCUPIED`, daily charge job posts a `charges` row per calendar day from `bed_allocations` → nursing rounds, med administration, OT bookings and diagnostics all post charges to the same `visit_id` → discharge initiated → pharmacy returns settled, interim invoices consolidated into a FINAL invoice → payment/claim → bed set `CLEANING` → housekeeping task auto-created → verified → bed `VACANT`.

### 5.3 Housekeeping flow
Trigger (discharge, transfer, soiling report, scheduled disinfection) → task created with type and priority → assigned to housekeeping staff → `IN_PROGRESS` → `DONE` → nurse verifies → bed released to `VACANT`. SLA timer per ward type; ICU beds cannot be re-allocated before verification.

### 5.4 Insurance claim flow
Policy captured at registration → eligibility check → pre-authorization requested for planned admission/surgery → approved amount recorded → charges accrue → at discharge the invoice splits `payer_amount` / `patient_amount` per contract → claim submitted with documents → queries handled → approval → settlement recorded, shortfall raised as a patient-receivable or written off via credit note.

### 5.5 Charge capture (new, replaces "temporary patient cart")
Every revenue module writes a `charges` row at the moment a service is rendered (consultation finalized, sample verified, study reported, drug dispensed, bed-day accrued, OT completed). Billing only reads `charges`; it never queries other modules. This removes the current cross-module billing coupling and makes interim IPD bills possible.

---

## 6. Suggested changes to the repository

| File | Change |
|---|---|
| `model/HMS.md` | Replace §3 role matrix with §4.3 here; replace §4 module list with the 18-module map; extend §8 core schema with the missing tables; replace §7 ER description with per-context diagrams; add the unified visit state machine to §13; add "Charge capture" before §20 billing workflow; add a Security §24 sub-section on break-glass access and append-only EMR |
| `model/HMS/Patient Management .md` | Add MRN policy, patient merge/duplicate handling, consents, access logging |
| `model/HMS/Staff Management.md` | Add Nurse, Cashier, Housekeeping, Insurance Desk, Auditor roles; shifts and attendance entities |
| `model/HMS/Doctor Management.md` | Add `doctor_schedules`/`slots`/`doctor_leaves` schema; define slot-holding and overbooking rules |
| `model/HMS/Appointment & Queue Management.md` | Introduce the `visits` entity; define token generation, no-show and cancellation policy |
| `model/HMS/Consultation & EMR ….md` | Add vitals, allergies, ICD-10 diagnoses, notes + amendments, referrals, follow-ups; state that finalized notes are append-only |
| `model/HMS/Laboratory & Diagnostics.md` | Split radiology out into a new `Radiology & Imaging.md`; add sample rejection, result verification and critical-value alerting |
| `model/HMS/Pharmacy & Dispensing.md` + `Pharmacy Workflow .md` | Merge into one document; add FEFO override logging, substitutions, returns, narcotic register |
| `model/HMS/Inventory & Supply Chain .md` | Make `stock_ledger` the single source of truth; add landed cost, transfers, expiry write-off approvals |
| `model/HMS/Billing & Financials.md` | Add `charges`, `invoice_items`, deposits, interim vs final invoices, refunds/credit notes, payer split |
| `model/HMS/Workflows.md` | Replace with the revised lifecycle, plus IPD, housekeeping, OT and claim flows |
| **New** `model/HMS/Inpatient & Bed Management.md` | Section 3.11 + 5.2 + 5.3 |
| **New** `model/HMS/Insurance & Claims.md` | Section 3.15 + 5.4 |
| **New** `model/HMS/Emergency & Triage.md` | Section 3.6 |
| **New** `model/HMS/Operation Theatre.md` | Section 3.10 |
| **New** `model/HMS/Access Control.md` | Section 4 in full (permission codes, scopes, matrix, sensitive actions) |

---

## 7. Suggested build order (when implementation starts)

1. IAM + Organisation master data + access-control enforcement.
2. Patient, Doctor, Appointment & Queue (first usable slice: register → book → check in).
3. Consultation & EMR + prescriptions.
4. Laboratory, then Radiology.
5. Pharmacy + Inventory (shared stock ledger).
6. Charge capture + Billing + payments.
7. Inpatient, beds, housekeeping, discharge.
8. Insurance & claims.
9. Notifications, reporting, audit dashboards.
