# Module 15 — Billing & Financials

## Purpose
Charge capture from every revenue module, interim and final invoicing, payments, refunds, credit notes and deposits.

## Entities
| Table | Key columns |
|---|---|
| charges | visit_id, service_id, source_module, source_ref_id, qty, unit_price, discount, tax, net_amount, status(`UNBILLED/BILLED/CANCELLED`) |
| invoices | visit_id, patient_id, invoice_no, type(`INTERIM/FINAL`), subtotal, discount_total, tax_total, payer_amount, patient_amount, total, status(`DRAFT/ISSUED/PARTIALLY_PAID/PAID/VOID`) |
| invoice_items | invoice_id, charge_id, description, qty, unit_price, amount |
| payments | invoice_id, amount, method(`CASH/CARD/UPI/BANK/INSURANCE/WALLET`), transaction_ref, received_by, received_at |
| refunds | payment_id, amount, reason, requested_by, approved_by |
| credit_notes | invoice_id, amount, reason, approved_by |
| deposits | visit_id, amount, received_by, adjusted_invoice_id |
| discounts | invoice_id/charge_id, type, amount_or_percent, reason, approved_by |

## Charge capture (replaces the "temporary patient cart")
Every revenue module writes a `charges` row the moment a service is rendered: consultation finalised, lab result verified, imaging study reported, drug dispensed, bed-day accrued, OT completed. Billing reads only `charges` and never queries other modules. This decouples billing and makes interim inpatient bills possible at any point.

## Workflows
1. **Accrue** — charges collect against the open `visit_id`.
2. **Interim invoice** (IPD) — issued on demand; already-billed charges are marked `BILLED` and cannot be re-billed.
3. **Final invoice** — at discharge or OPD completion: discounts applied, taxes computed, payer/patient split resolved from the insurance module.
4. **Payment** — full or partial; deposits auto-adjusted; receipt generated.
5. **Refund / void** — cashier requests, admin approves; a void always issues a credit note. Invoices are never deleted or silently edited.

## Role access
| Action | Cashier | Hospital Admin | Receptionist | Pharmacist | Insurance Desk | Doctor | Patient | Auditor |
|---|---|---|---|---|---|---|---|---|
| View charges | Read | Read | Create manual | Auto-posted | Read | Read (own patients) | Read (own) | Read |
| Create invoice | Full | Full | Create | Create (pharmacy POS) | Read | – | – | Read |
| Collect payment | Full | Read | Create | Create | Record insurance receipt | – | Pay (own) | Read |
| Discount | Request | Approve | – | – | Contract-based | – | – | Read |
| Refund / void / credit note | Request | Approve | – | – | – | – | – | Read |
| Financial reports | Department | Facility | – | Pharmacy | Claims | Own earnings | – | Full |

## Rules
- One charge can appear on exactly one invoice.
- Issued invoices are immutable; corrections are credit notes or new invoices.
- Discounts above a threshold need admin approval and a recorded reason.
- Every money amount is stored in integer minor units with a currency code.
- Daily cash reconciliation per cashier is mandatory at shift close.

## Dependencies
Upstream: all revenue modules, Master data (price list). Downstream: Insurance & Claims, Reporting.
