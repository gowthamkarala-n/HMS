# Module 01 — Identity & Access Management (IAM)

## Purpose
Authentication, role assignment, permission evaluation and session control for every user of the HMS.

## Entities
| Table | Key columns |
|---|---|
| users | username, email, phone, password_hash, status(`ACTIVE/INACTIVE/LOCKED`), mfa_enabled, last_login_at |
| roles | code, name, is_system |
| permissions | code (`module.entity.action`), description |
| role_permissions | role_id, permission_id, scope(`own/assigned/department/facility/global`) |
| user_roles | user_id, role_id, facility_id, department_id, valid_from, valid_to |
| sessions | user_id, refresh_token_hash, ip, user_agent, expires_at, revoked_at |
| password_resets | user_id, token_hash, expires_at, used_at |

Conventions: UUID ids, `created_at/updated_at/created_by/updated_by`, `facility_id` on scoped rows, soft delete on master data.

## Access model
Access is `role → permission → scope`, never role → module.

- Permission code: `module.entity.action` — e.g. `pharmacy.dispense.create`, `emr.note.amend`.
- Actions: `read`, `create`, `update`, `delete`, `approve`, `export`.
- Scopes: `own`, `assigned`, `department`, `facility`, `global`.

Roles (15): Super Admin, Hospital Admin, Doctor, Nurse, Receptionist, Cashier, Pharmacist, Lab Technician, Lab Manager, Radiologist, Inventory Manager, Ward/Housekeeping, Insurance Desk, Auditor, Patient.

## Workflows
1. **Login** — credentials verified (BCrypt/Argon2), access token (15 min) + refresh token (7 days) issued as HttpOnly cookies, MFA challenge when enabled.
2. **Authorization** — every request resolves the user's active `user_roles` for the current facility, expands to permissions with scopes, and the scope is applied as a row filter.
3. **Role assignment** — Admin grants a role scoped to a facility/department with validity dates; expiry is automatic.
4. **Lockout** — N failed attempts locks the account; unlock is an audited admin action.

## Role access
| Action | Super Admin | Hospital Admin | Auditor | Others |
|---|---|---|---|---|
| Manage users | Full | Full | Read | – |
| Manage roles & permissions | Full | – | Read | – |
| Assign roles | Full | Full | Read | – |
| View sessions / revoke | Full | Full | Read | Own only |

## Rules
- Roles are stored in a dedicated `user_roles` table — never as a column on a user or profile row.
- No user may grant a permission they do not themselves hold.
- Every permission change is written to `audit_logs`.
- Patient accounts have `own` scope only, on every module.

## Dependencies
None upstream. Every other module depends on this one.
