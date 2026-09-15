# Module 17 — Notification & Communication

## Purpose
Event-driven messaging to patients and staff across email, SMS, push and in-app channels.

## Entities
| Table | Key columns |
|---|---|
| notification_templates | code, channel(`EMAIL/SMS/PUSH/INAPP`), subject, body, locale |
| notifications | user_id/patient_id, template_code, payload, status(`PENDING/SENT/FAILED/READ`), scheduled_at, sent_at |
| delivery_logs | notification_id, provider, provider_ref, status, error |
| user_notification_preferences | user_id, event_code, channel, enabled, quiet_hours |

## Event catalogue
| Event | Recipients | Channels |
|---|---|---|
| Appointment booked / rescheduled / cancelled | Patient, Doctor | SMS, Email, In-app |
| Appointment reminder (T-24h, T-2h) | Patient | SMS, Push |
| Queue token called | Patient | In-app, display board |
| Lab / imaging report ready | Patient, Ordering doctor | Email, In-app |
| Critical lab value | Ordering doctor, Duty nurse | Push + phone escalation |
| Prescription ready for pickup | Patient | SMS |
| Admission, transfer, discharge | Patient contact | SMS, Email |
| Invoice issued / payment received | Patient | Email |
| Pre-auth approved / claim queried | Insurance desk | In-app, Email |
| Low stock / expiring batch | Inventory Manager, Pharmacist | In-app, Email |
| Pending doctor approval | Hospital Admin | In-app |
| Housekeeping task assigned / SLA breach | Housekeeping, Nurse | Push, In-app |

## Workflow
Domain module publishes an event → notification service resolves recipients and their preferences → renders the template → queues per channel → provider dispatch (Email/SMS/Push) → delivery status written to `delivery_logs` → failures retried with backoff, then escalated.

## Role access
| Action | Super Admin | Hospital Admin | All staff | Patient | Auditor |
|---|---|---|---|---|---|
| Templates | Full | Create/Update | Read | – | Read |
| Send manual message | Full | Create | Within own module | – | Read |
| Delivery logs | Full | Read | Read (own sends) | – | Read |
| Preferences | Full | Read | Update (own) | Update (own) | Read |

## Rules
- Clinical content is never sent in an SMS body; the message links to the portal.
- Quiet hours apply to non-critical notifications only; critical alerts always go through.
- Every notification records the template version used.

## Dependencies
Upstream: all modules (event producers), IAM. Downstream: none.
