# Workflows

### Role Hierarchy

```mermaid
graph TD
    SA[Super Admin] --> A[Admin]
    A --> DOC[Doctor]
    A --> STAFF[Staff: Nurse/Receptionist/Lab/Inventory]
    A --> PHARM[Pharmacist]
    DOC --> PAT[Patient]
    STAFF --> PAT
```

### 3. Role Permissions Matrix

| Module | Super Admin | Admin | Doctor | Receptionist | Pharmacist | Lab Staff | Inventory | Patient |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **System Config** | Full | Read | - | - | - | - | - | - |
| **User/Staff Mgmt** | Full | Full | - | - | - | - | - | - |
| **Doctor Approval** | Full | Full | Read(Own) | - | - | - | - | - |
| **Appointments** | Read | Full | Manage(Own) | Book/Cancel | - | - | - | Book(Own) |
| **Consultations** | Read | Read | Full (Own) | - | - | - | - | Read(Own) |
| **Pharmacy/Inventory** | Read | Read | Read | - | Full | - | Full | Read(Own) |
| **Laboratory** | Read | Read | Req/View | - | - | Full | - | View(Own) |
| **Billing/Payments** | Read | Full | View | Generate | Generate | - | - | Pay/View |

### 🔄 Staff Status State Machine

```mermaid
stateDiagram-v2
    [*] --> ONBOARDING: HR Creates Profile
    ONBOARDING --> PENDING_CREDENTIALING: Docs Uploaded (Doctors)
    ONBOARDING --> ACTIVE: Docs Verified (Non-Clinical)
    PENDING_CREDENTIALING --> ACTIVE: Medical Board Approves
    PENDING_CREDENTIALING --> REJECTED: Invalid License
    ACTIVE --> ON_LEAVE: Approved Leave
    ON_LEAVE --> ACTIVE: Leave Ends
    ACTIVE --> SUSPENDED: Malpractice / Investigation
    SUSPENDED --> ACTIVE: Cleared by Admin
    ACTIVE --> INACTIVE: Resigned / Terminated
    INACTIVE --> [*]
```

---

### 🔄 Doctor Account & Status State Machine

```mermaid
stateDiagram-v2
    [*] --> ONBOARDING: HR Creates Profile
    ONBOARDING --> PENDING_CREDENTIALING: Docs Uploaded
    PENDING_CREDENTIALING --> ACTIVE: Medical Board Approves
    PENDING_CREDENTIALING --> REJECTED: Invalid License
    ACTIVE --> ON_LEAVE: Approved Sabbatical/Leave
    ON_LEAVE --> ACTIVE: Return to Duty
    ACTIVE --> SUSPENDED: Malpractice / License Expired
    SUSPENDED --> ACTIVE: Cleared / License Renewed
    ACTIVE --> INACTIVE: Resigned / Terminated
    INACTIVE --> [*]
```

---

### 🔄 Patient Visit State Machine (Status Lifecycle)

To ensure smooth queue management and reporting, every patient visit moves through strict states:

```mermaid
stateDiagram-v2
    [*] --> SCHEDULED: Appointment Booked
    SCHEDULED --> CHECKED_IN: Receptionist Checks In
    CHECKED_IN --> TRIAGE: Nurse Calls Patient
    TRIAGE --> READY_FOR_DOCTOR: Vitals Recorded
    READY_FOR_DOCTOR --> IN_CONSULTATION: Doctor Starts Session
    IN_CONSULTATION --> PENDING_DIAGNOSTICS: Orders Placed (Lab/Rad)
    PENDING_DIAGNOSTICS --> IN_CONSULTATION: Results Reviewed
    IN_CONSULTATION --> PENDING_BILLING: Consultation Finalized
    PENDING_BILLING --> DISCHARGED: Payment Completed
    DISCHARGED --> [*]

    CHECKED_IN --> NO_SHOW: Patient didn't arrive
    IN_CONSULTATION --> ADMITTED: Doctor orders Inpatient
```

---

## 🔄 Pharmacy (Prescription & Dispensing State Machine)

```mermaid
stateDiagram-v2
    [*] --> PRESCRIBED: Doctor signs e-Prescription
    PRESCRIBED --> CDSS_REVIEW: Automated safety check triggered
    CDSS_REVIEW --> QUEUED: Cleared (No alerts)
    CDSS_REVIEW --> FLAGGED: Alert raised (Allergy/Interaction)
    FLAGGED --> QUEUED: Doctor overrides with reason
    FLAGGED --> CANCELLED: Doctor changes prescription

    QUEUED --> PREPARING: Pharmacist clicks "Prepare"
    PREPARING --> ALLOCATED: FEFO batch selected & confirmed
    ALLOCATED --> DISPENSED: Medicine handed to OPD patient
    ALLOCATED --> UNIT_DOSE_READY: IPD: Unit-dose packet prepared

    UNIT_DOSE_READY --> DELIVERED_TO_WARD: Sent to ward
    DELIVERED_TO_WARD --> ADMINISTERED: Nurse scans & confirms (eMAR)
    DELIVERED_TO_WARD --> RETURNED: Unused, sent back to pharmacy
    RETURNED --> QC_HOLD: Awaiting pharmacist inspection
    QC_HOLD --> RESTOCKED: Cleared, returned to inventory
    QC_HOLD --> DESTROYED: Failed QC, sent to destruction

    DISPENSED --> RETURNED_OPD: Patient returns unopened medicine
    RETURNED_OPD --> RESTOCKED: QC passed
    RETURNED_OPD --> DESTROYED: QC failed

    DISPENSED --> [*]
    ADMINISTERED --> [*]
    DESTROYED --> [*]
    CANCELLED --> [*]
```

---

### 🔄 Lab Order State Machine

```mermaid
stateDiagram-v2
    [*] --> ORDERED: Doctor places order & auto-bills
    ORDERED --> COLLECTED: Nurse scans wristband + tube
    ORDERED --> CANCELLED: Doctor cancels / Patient refuses

    COLLECTED --> RECEIVED_IN_LAB: Lab scans sample at bench
    COLLECTED --> REJECTED: Sample hemolyzed / insufficient volume

    RECEIVED_IN_LAB --> IN_PROGRESS: Loaded into analyzer
    IN_PROGRESS --> VERIFIED: Pathologist reviews & signs
    IN_PROGRESS --> RERUN: Analyzer error / invalid result

    VERIFIED --> COMPLETED: Published to EMR & Patient App
    REJECTED --> COLLECTED: Recollect new sample

    COMPLETED --> [*]
    CANCELLED --> [*]
```

---

## 🔄 Inventory State Machines

### Consumable / Batch State Machine

```mermaid
stateDiagram-v2
    [*] --> ORDERED: PO Generated
    ORDERED --> IN_TRANSIT: Vendor Dispatched
    IN_TRANSIT --> RECEIVED: GRN Created, Batch/Expiry Logged
    RECEIVED --> ACTIVE: Available for Ward Issue (FEFO)
    ACTIVE --> QUARANTINED: Near Expiry or Recall Triggered
    QUARANTINED --> DESTROYED: Dual-Witness Sign-off
    DESTROYED --> [*]

    ACTIVE --> ISSUED_TO_WARD: Ward Indent Fulfilled
    ISSUED_TO_WARD --> CONSUMED: Scanned at Patient Bedside (Auto-Billed)
    CONSUMED --> [*]
```

### Reusable Asset State Machine

```mermaid
stateDiagram-v2
    [*] --> IN_STORE: Asset Commissioned & Tagged
    IN_STORE --> ISSUED: Assigned to Ward/Patient
    ISSUED --> IN_USE: Actively being used
    IN_USE --> DIRTY: Patient Discharged / Procedure Complete
    DIRTY --> CLEANING: Sent to CSSD (Sterilization)
    CLEANING --> IN_STORE: Cleared for reuse

    IN_USE --> FAULTY: Equipment Malfunction
    FAULTY --> UNDER_MAINTENANCE: Biomed Work Order Created
    UNDER_MAINTENANCE --> IN_STORE: Repaired & Calibrated
    UNDER_MAINTENANCE --> OBSOLETE: Beyond Economic Repair
    OBSOLETE --> SCRAPPED: Dual-Witness Write-off
    SCRAPPED --> [*]
```

---

### 🔄 Invoice & Payment State Machine

```mermaid
stateDiagram-v2
    [*] --> PENDING: Clinical Event Triggered (Zero-Click)
    PENDING --> BILLED: Invoice Generated / Discharge

    BILLED --> PARTIALLY_PAID: Insurance approved partial / Patient paid deposit
    PARTIALLY_PAID --> PAID: Final settlement completed
    BILLED --> PAID: Full payment received

    PAID --> REFUNDED: Reversing Entry created (Medicine return / Error)
    REFUNDED --> PAID: Net settlement adjusted

    BILLED --> VOID: Invoice cancelled before payment (Requires Admin Approval)

    PAID --> [*]
    VOID --> [*]
```

---
