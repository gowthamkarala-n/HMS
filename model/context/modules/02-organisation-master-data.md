# Module 02 — Organisation & Master Data

## Purpose
Facilities, departments, the service/price catalogue and clinical code sets that every other module references.

## Entities
| Table | Key columns |
|---|---|
| facilities | code, name, address, timezone, is_active |
| departments | facility_id, code, name, head_staff_id |
| services | code, name, department_id, category(`CONSULT/LAB/IMAGING/PROCEDURE/BED/PHARMACY`), is_active |
| price_list | service_id, payer_type, amount, currency, valid_from, valid_to |
| tax_rates | code, percent, valid_from, valid_to |
| icd_codes | code, title, version |
| loinc_codes | code, title, unit |
| uoms | code, name, base_factor |
| rooms_master (ref) | see Inpatient module |

## Workflows
1. **Service onboarding** — create service → attach to department → publish price with `valid_from`.
2. **Price revision** — never edit an existing price row; close it with `valid_to` and insert a new effective-dated row. Historical invoices stay reproducible.
3. **Payer-specific pricing** — a `price_list` row per payer type (SELF / INSURER / CORPORATE / GOVT).

## Role access
| Entity | Super Admin | Hospital Admin | Clinical & operational roles | Auditor |
|---|---|---|---|---|
| Facilities / departments | Full | Create/Update | Read | Read |
| Services | Full | Create/Update + Approve | Read | Read |
| Price list | Full | Create/Update + Approve | Read | Read |
| Code sets (ICD/LOINC) | Full | Update | Read | Read |

## Rules
- Every billable action in the system resolves to exactly one `service_id`.
- Prices are effective-dated; retroactive edits are forbidden.
- Deactivating a service does not delete it — existing charges keep referencing it.
- Money is stored as integer minor units with an explicit currency.

## Dependencies
Upstream: IAM. Downstream: Billing, Laboratory, Radiology, Pharmacy, Inpatient, OT.
