# Module 13 — Pharmacy & Dispensing

Merges the repo's separate "Pharmacy & Dispensing" and "Pharmacy Workflow" documents.

## Purpose
Medicine catalogue, batch and expiry control, FEFO dispensing against e-prescriptions, walk-in POS sales, returns and narcotic control.

## Entities
| Table | Key columns |
|---|---|
| medicines | item_id, generic_name, brand, strength, form, schedule(`OTC/H/H1/NARCOTIC`), requires_prescription |
| medicine_batches | medicine_id, batch_no, expiry_date, mrp, cost, qty_on_hand, location |
| pharmacy_orders | prescription_id (nullable for walk-in), visit_id, status(`QUEUED/PARTIAL/DISPENSED/CANCELLED`) |
| dispenses | order_id, dispensed_by, dispensed_at, invoice_id |
| dispense_items | dispense_id, medicine_id, batch_id, qty, unit_price, substitution_of |
| pharmacy_returns | dispense_item_id, qty, reason, approved_by, restocked |
| fefo_overrides | dispense_item_id, reason, approved_by |
| narcotic_register | dispense_item_id, prescriber_licence, balance_after, witnessed_by |

## Workflows
1. **Prescription arrives** — doctor finalises an e-prescription; it lands in the pharmacy queue for that visit.
2. **Review** — pharmacist checks interactions, allergy conflicts and stock. Substitution to an equivalent generic is allowed with a recorded reason.
3. **FEFO allocation** — the system auto-allocates batches by earliest expiry. Any manual deviation creates a `fefo_overrides` row with a reason and admin approval.
4. **Dispense** — stock deducted through the inventory `stock_ledger`; batch, quantity and price captured per item; partial dispensing supported.
5. **Charge** — a charge posts to the visit. Outpatients pay at the counter; inpatients accrue to the final bill.
6. **Return** — unused inpatient medicines are returned at discharge; only unexpired, intact batches are restocked.
7. **Narcotics** — schedule drugs require prescriber licence, a witness and a running register balance.

## Role access
| Action | Pharmacist | Hospital Admin | Doctor | Nurse | Inventory | Cashier | Patient | Auditor |
|---|---|---|---|---|---|---|---|---|
| Medicine & batch master | Full | Create/Update | Read | – | Full | – | – | Read |
| Prescription queue | Read | Read | Read (own) | Read (assigned) | – | – | Read (own) | Read |
| Dispense | Full | Read | – | – | – | – | – | Read |
| FEFO override | Request | Approve | – | – | – | – | – | Read |
| Returns | Create | Approve | – | Request (ward) | Read | – | – | Read |
| POS invoice | Create | Read | – | – | – | Read | Read (own) | Read |
| Narcotic register | Create | Read | Read (own) | Witness | – | – | – | Read |

## Rules
- Expired batches can never be allocated, overridden or sold.
- Prescription-only medicines cannot be dispensed without a valid prescription reference.
- Stock changes only ever happen through the `stock_ledger`; the batch quantity is a projection of it.
- Dispensing is blocked when the allocated quantity exceeds the prescribed quantity.

## Dependencies
Upstream: EMR, Inventory, Master data. Downstream: Billing, Inpatient.
