# Module 14 — Inventory & Supply Chain

## Purpose
Items and suppliers, purchase orders, goods receipt, batch-level stock with a single ledger of truth, transfers, adjustments and expiry write-offs.

## Entities
| Table | Key columns |
|---|---|
| items | code, name, category(`DRUG/CONSUMABLE/ASSET/LINEN/REAGENT`), uom, reorder_level, is_batch_tracked |
| suppliers | name, contact, email, gst_no, address, rating |
| supplier_items | supplier_id, item_id, price, lead_time_days |
| purchase_orders | supplier_id, po_no, status(`DRAFT/SENT/PARTIAL/RECEIVED/CANCELLED`), expected_at, approved_by |
| po_items | po_id, item_id, qty, unit_price |
| grns | po_id, received_by, supplier_invoice_no, received_at |
| grn_items | grn_id, item_id, batch_no, expiry, qty, unit_cost, landed_cost |
| stock_ledger | item_id, batch_id, location_id, txn_type, qty_delta, ref_type, ref_id, at |
| stock_transfers | from_location, to_location, item_id, batch_id, qty, approved_by |
| stock_adjustments | item_id, batch_id, qty_delta, reason, approved_by |
| expiry_writeoffs | batch_id, qty, approved_by, disposal_ref |

`stock_ledger.txn_type`: `GRN / ISSUE / DISPENSE / RETURN / ADJUST / TRANSFER / EXPIRY_WRITEOFF`.

## Workflows
1. **Reorder** — stock below `reorder_level` (or an admin decision) raises a purchase requisition.
2. **Purchase order** — PO drafted, approved by admin above a value threshold, sent to the supplier.
3. **Goods receipt** — GRN captures batch number, expiry, quantity and unit cost; landed cost computed with freight and taxes; every line posts a `GRN` ledger row.
4. **Issue / transfer** — stock moves to pharmacy, ward, lab or OT locations; each move is a ledger row.
5. **Expiry control** — alerts at 90/60/30 days; expired stock is quarantined and written off with approval and a disposal reference.
6. **Stock take** — periodic physical count produces `ADJUST` rows with reasons; variance beyond a threshold requires dual approval.

## Role access
| Action | Inventory Manager | Hospital Admin | Pharmacist | Lab Manager | Nurse | Housekeeping | Auditor |
|---|---|---|---|---|---|---|---|
| Items & suppliers | Full | Create/Update | Read | Read | – | Read | Read |
| Purchase orders | Full | Approve | Request | Request | – | – | Read |
| GRN | Full | Read | Create (pharmacy) | Create (lab) | – | – | Read |
| Stock ledger | Full | Read | Create via dispense | Create via usage | Request issue | Create (linen) | Read |
| Transfers | Full | Approve | Request | Request | Request | – | Read |
| Adjustments / write-offs | Request | Approve | Request | Request | – | – | Read |

## Rules
- `stock_ledger` is append-only and is the only way stock changes; on-hand quantities are derived.
- A batch-tracked item cannot be received without batch number and expiry.
- Negative stock is never allowed; an issue that would go negative is rejected.
- Write-offs above a configured value require dual approval.

## Dependencies
Upstream: Organisation. Downstream: Pharmacy, Laboratory, OT, Inpatient, Billing (cost of goods).
