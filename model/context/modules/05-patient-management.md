# Module 05 — Patient Management

## Purpose
Patient identity: registration, MRN, demographics, contacts, consents, documents and duplicate handling.

## Entities
| Table | Key columns |
|---|---|
| patients | user_id (nullable), mrn (unique per facility), first_name, last_name, dob, gender, blood_group, phone, email, address, is_deceased |
| patient_contacts | patient_id, relation, name, phone, is_emergency |
| patient_consents | patient_id, type(`TREATMENT/DATA_SHARING/SURGERY/TELEMEDICINE`), granted_at, revoked_at, document_url |
| patient_documents | patient_id, doc_type(`ID/INSURANCE/REFERRAL/EXTERNAL_REPORT`), file_url, uploaded_by |
| patient_merges | source_patient_id, target_patient_id, merged_by, merged_at, reason |
| access_logs | user_id, patient_id, purpose, at |

## Workflows
1. **Registration** — walk-in or online; duplicate check on phone + name + DOB before MRN allocation.
2. **Portal linking** — an existing patient record can be linked to a self-service `users` account after OTP verification.
3. **Merge** — duplicates are merged, not deleted: all visits, invoices and records repoint to the target MRN and the merge is audited.
4. **Consent** — capture at registration and per procedure; revocation is recorded, never overwritten.

## Role access
| Action | Hospital Admin | Receptionist | Doctor / Nurse | Lab / Radiology | Cashier / Insurance | Patient | Auditor |
|---|---|---|---|---|---|---|---|
| Register patient | Full | Full | – | – | – | Self-register | Read |
| Demographics | Full | Create/Update | Read (assigned) | Read (assigned) | Read | Read/Update (own) | Read |
| Contacts & consents | Full | Create/Update | Read | – | Read | Read/Update (own) | Read |
| Documents | Full | Upload | Read (assigned) | Read (assigned) | Read insurance | Upload (own) | Read |
| Merge records | Approve | Request | – | – | – | – | Read |

## Rules
- No hard delete of a patient. Deactivate or merge only.
- Every read of a patient record outside the reader's `assigned` scope writes an `access_logs` row with a mandatory purpose (break-glass).
- MRN format is facility-prefixed and never reused.
- Deceased flag blocks new appointment booking.

## Dependencies
Upstream: IAM. Downstream: every clinical and financial module.
