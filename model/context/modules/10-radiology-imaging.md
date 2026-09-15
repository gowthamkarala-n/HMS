# Module 10 — Radiology & Imaging

## Purpose
Imaging orders, modality scheduling, study capture and radiologist reporting. Separated from Laboratory because scheduling, equipment and reporting differ fundamentally.

## Entities
| Table | Key columns |
|---|---|
| modalities | code(`XRAY/CT/MRI/USG/MAMMO`), room_id, status, maintenance_until |
| imaging_services | service_id, modality_id, prep_instructions, duration_minutes, contrast_required |
| imaging_orders | visit_id, consultation_id, ordered_by, priority, clinical_history, status |
| imaging_studies | order_id, modality_id, scheduled_at, performed_by, performed_at, accession_no, dicom_uid, image_urls |
| imaging_reports | study_id, findings, impression, radiologist_id, signed_at, addendum_of |

`imaging_orders.status`: `ORDERED → SCHEDULED → PERFORMED → REPORTED → SIGNED`, plus `CANCELLED`.

## Workflows
1. **Order** — doctor orders a study with clinical history; contrast studies require a creatinine check and consent.
2. **Schedule** — reception or radiology desk books a modality time slot, honouring prep and duration.
3. **Perform** — technologist performs the study, links images/DICOM UID, marks `PERFORMED`.
4. **Report** — radiologist writes findings and impression and signs. Addenda are new rows referencing the original.
5. **Release** — signed report routes to the ordering doctor and the patient portal; a charge is posted per study.

## Role access
| Action | Radiologist | Technologist (Lab Tech role) | Doctor | Reception | Nurse | Patient | Auditor |
|---|---|---|---|---|---|---|---|
| Imaging catalogue | Read | Read | Read | Read | Read | – | Read |
| Orders | Read | Read | Create (assigned) | Schedule | Read (assigned) | Read (own) | Read |
| Studies / images | Full | Create/Update | Read (assigned) | Read status | Read (assigned) | Read (own) | Read |
| Reports | Create/Update/Sign | – | Read (assigned) | – | Read (assigned) | Read (own, signed) | Read |
| Addendum | Approve (own) | – | – | – | – | – | Read |

## Rules
- Unsigned reports are never released outside radiology.
- Accession number is unique and immutable.
- Radiation dose and contrast usage are recorded per study.
- Modality under maintenance cannot be scheduled.

## Dependencies
Upstream: EMR, Master data, Organisation. Downstream: Billing, Notifications.
