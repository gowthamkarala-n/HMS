# Module 16 — Insurance & Claims

## Purpose
Payers and contracts, patient policies, pre-authorisation, claim submission and settlement, and the payer/patient split on invoices. Entirely missing from the current repo specification.

## Entities
| Table | Key columns |
|---|---|
| payers | name, type(`INSURER/TPA/CORPORATE/GOVT`), contact, portal_url |
| payer_contracts | payer_id, price_list_id, discount_terms, tds_percent, valid_from, valid_to |
| patient_policies | patient_id, payer_id, policy_no, valid_from, valid_to, sum_insured, balance |
| preauthorizations | visit_id, policy_id, requested_amount, approved_amount, status(`REQUESTED/APPROVED/PARTIAL/REJECTED`), reference_no |
| claims | invoice_id, policy_id, claim_no, claimed_amount, status(`DRAFT/SUBMITTED/QUERIED/APPROVED/REJECTED/SETTLED`) |
| claim_items | claim_id, invoice_item_id, claimed_amount, allowed_amount, disallowed_reason |
| claim_documents | claim_id, doc_type, file_url |
| settlements | claim_id, settled_amount, tds, settled_at, utr |

## Workflows
1. **Eligibility** — policy captured at registration; validity and balance checked before a cashless admission.
2. **Pre-authorisation** — planned admission or surgery: request with estimate and clinical justification → payer approves an amount → recorded against the visit.
3. **Accrual** — charges use the payer contract's price list; the invoice splits `payer_amount` and `patient_amount`.
4. **Claim** — final invoice plus discharge summary, reports and bills submitted → payer queries handled → approval.
5. **Settlement** — settled amount and TDS recorded against the claim; any shortfall becomes a patient receivable or is written off through a credit note.

## Role access
| Action | Insurance Desk | Hospital Admin | Cashier | Receptionist | Doctor | Patient | Auditor |
|---|---|---|---|---|---|---|---|
| Payers & contracts | Read | Full | Read | Read | – | – | Read |
| Patient policies | Full | Read | Read | Create/Update | Read (assigned) | Read (own) | Read |
| Pre-authorisation | Full | Read | Read | Request | Provide justification | Read (own) | Read |
| Claims | Full | Read | Read | – | – | Read (own) | Read |
| Settlements | Create | Approve write-off | Read | – | – | Read (own) | Read |

## Rules
- Cashless treatment beyond the approved pre-auth amount requires an enhancement request before discharge.
- Disallowed claim lines must carry a reason and are reconciled against the patient's receivable.
- Policy balance is decremented only on settlement, never on submission.
- Corporate and government schemes use their own contracted price list, not the self-pay list.

## Dependencies
Upstream: Patient, Billing, Inpatient, Master data. Downstream: Reporting.
