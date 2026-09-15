# Module 09 — Laboratory & Diagnostics

## Purpose
Test catalogue, order intake, sample lifecycle, result entry and verification, critical-value alerting.

## Entities
| Table | Key columns |
|---|---|
| lab_tests | service_id, loinc_code, specimen_type, normal_range_low/high, unit, tat_minutes, method |
| lab_panels / lab_panel_items | panel → constituent tests |
| lab_orders | visit_id, consultation_id, ordered_by, priority(`ROUTINE/URGENT/STAT`), status |
| lab_order_items | lab_order_id, lab_test_id, status |
| lab_samples | lab_order_id, barcode, specimen_type, collected_by, collected_at, received_at, rejected_reason |
| lab_results | lab_order_item_id, value, unit, flag(`LOW/NORMAL/HIGH/CRITICAL`), entered_by, verified_by, verified_at |
| critical_result_alerts | lab_result_id, notified_doctor_id, notified_at, acknowledged_at |

`lab_orders.status`: `ORDERED → COLLECTED → IN_PROCESS → COMPLETED → VERIFIED`, plus `CANCELLED`.

## Workflows
1. **Order** — doctor orders during consultation; order enters the lab queue with priority.
2. **Collection** — technician collects the sample, prints a barcode, marks `COLLECTED`. Bad samples are rejected with a reason and re-collection is requested.
3. **Processing & entry** — results entered against each order item, auto-flagged against reference ranges.
4. **Verification** — lab manager (or authorised senior technician) verifies before release. Only verified results reach the doctor and the patient portal.
5. **Critical value** — a `CRITICAL` flag notifies the ordering doctor immediately and requires acknowledgement.
6. **Charge** — a charge is posted per test at order confirmation (outpatient: prepaid; inpatient: accrued to the visit).

## Role access
| Action | Lab Technician | Lab Manager | Doctor | Nurse | Reception | Patient | Auditor |
|---|---|---|---|---|---|---|---|
| Test catalogue | Read | Create/Update | Read | Read | Read | – | Read |
| Orders | Read/Update status | Full | Create (assigned), Read | Read (assigned) | Read status | Read (own) | Read |
| Samples | Full | Full | Read | Read | – | – | Read |
| Result entry | Create/Update | Create/Update | – | – | – | – | Read |
| Result verification | – | Approve | – | – | – | – | Read |
| Report download | Read | Read | Read (assigned) | Read (assigned) | – | Read (own, verified) | Read |

## Rules
- Unverified results are never visible to patients.
- A result may be corrected only by issuing a corrected report; the original is retained.
- Barcode is unique and links sample → order → result end to end.
- Cancelling an order after collection requires a reason and reverses the charge.

## Dependencies
Upstream: EMR, Master data. Downstream: Billing, Notifications.
