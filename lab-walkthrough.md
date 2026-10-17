# Lab walkthrough

This document records how each part of the project was configured in SailPoint IdentityIQ 8.1. Menu names can differ slightly between patch levels.

## 1. Environment

| Component | Notes |
|---|---|
| IdentityIQ | 8.1, deployed on Apache Tomcat |
| Database | MySQL |
| Initialisation | `init.xml` and `init-lcm.xml` imported through the IIQ console (the second one enables Lifecycle Manager) |
| Data folder | `C:\SailPointData\` |

## 2. Source data

| File | Used by | Purpose |
|---|---|---|
| `hr_source_data.txt` | `HR-Source` | Authoritative HR feed. Baseline is in `data/hr_source_data.baseline.txt` |
| `corp_app_data.txt` | `Corp-App` | Same layout, edited independently so access can be changed without altering the HR record. Final state is in `data/corp_app_data.txt` |

Columns: `EmployeeID, FirstName, LastName, Email, Department, Manager, Status, JobTitle`.

Designed test conditions in the baseline:

- `M001` (John Doe) manages everyone else, so he is the certifier and manager correlation target.
- The `Status` column holds Joiner, Mover and Leaver values to drive lifecycle tests.
- Alice Smith is in IT but has the job title *HR Specialist*, which creates both an SoD conflict and an outlier.
- Frank's surname is lowercase (`jatt`) in the source.

## 3. Identity attributes

**Gear icon → Identity Mappings.** Added searchable attributes:

| Identity attribute | Source column |
|---|---|
| `employeeId` | EmployeeID |
| `department` | Department |
| `jobTitle` | JobTitle |
| `hrStatus` | Status |

## 4. Applications

### HR-Source (authoritative)
- Type: DelimitedFile, **Authoritative Application** ticked
- Identity attribute `EmployeeID`, display attribute `Email`
- Correlation: `EmployeeID` → `employeeId`
- Manager correlation: `Manager` → `employeeId`
- Task: `Agg-HR` (Account Aggregation with *Refresh identity attributes*)

### Corp-App (entitlements)
- Type: DelimitedFile, not authoritative, owner `spadmin`
- Identity attribute `EmployeeID`, display attribute `Email`
- `Department` and `JobTitle` marked **Entitlement** and **Managed** (see [`exports/Corp-App.xml`](../exports/Corp-App.xml))
- Correlation: `EmployeeID` → `employeeId`
- Task: `Agg-Corp` (Account Aggregation with *Promote managed attributes*)

## 5. Entitlement catalog

Promoted from `Corp-App`; each entry was given a description, owner `spadmin` and the *Requestable* flag.

| Attribute | Values |
|---|---|
| Department | IT, HR, Management |
| JobTitle | Director, HR Coordinator, HR Specialist, Security Analyst, SysAdmin |

## 6. Roles (RBAC)

**IT roles** each contain one entitlement on `Corp-App`:

| Role | Entitlement |
|---|---|
| `IT-Dept-Access` | Department = IT |
| `HR-dept-access` | Department = HR |
| `Mgmt-Dept-Access` | Department = Management |
| `HR-Speciality-Access` | JobTitle = HR Specialist |
| `SysAdmin-Access` | JobTitle = SysAdmin |
| `SecAnalyst-Access` | JobTitle = Security Analyst |

**Business roles** use an assignment rule (Match List, identity attribute `Department` equals a value) and a **Required Role**:

| Role | Rule | Required role |
|---|---|---|
| `BR-IT-Staff` | Department = IT | `IT-Dept-Access` |
| `BR-HR-Staff` | Department = HR | `HR-dept-access` |
| `BR-Management` | Department = Management | `Mgmt-Dept-Access` |

Task: `Refresh-All` (Identity Refresh) with *Refresh assigned, detected roles*, *Provision assignments*, *Check active policies* and *Process events*.

## 7. Lifecycle events

Exports are in [`exports/lifecycle-events/`](../exports/lifecycle-events/).

| Event | Type | Trigger | Business process |
|---|---|---|---|
| Joiner | Create | New identity created | Lifecycle Event - Joiner |
| Mover | Attribute Change | `department` changes | Lifecycle Event - Email manager for all native changes |
| Leaver | Attribute Change | `hrStatus` becomes `Leaver` | Lifecycle Event - Leaver |

Role changes for movers come from the assignment rules during `Refresh-All`; the Mover event adds the notification step on top.

## 8. Access request

- Lifecycle Manager request access was already enabled after the LCM initialisation.
- The workflow `LCM Provisioning` uses an `approvalScheme` variable that defaults to `owner`, so approval goes to the entitlement owner.
- Bob Jones requested the *Security Analyst* entitlement (Request ID 2). The approval arrived as an owner approval work item. Because `Corp-App` is a file, the provisioning step becomes a manual change work item.

## 9. Certification

- Type: Manager certification for John Doe
- Contents: entitlements, additional entitlements, roles and policy violations
- Result: 4/4 identities and 8/8 items completed. Bob Jones's `SysAdmin-Access` was revoked and generated a remediation work item.
- Because the application is file based, the revoke was finished by clearing Bob's job title in `corp_app_data.txt` and re-running `Agg-Corp` and `Refresh-All`.

## 10. Policy

| Item | Value |
|---|---|
| Policy | `IT-vs-HR-SOD` (Role SOD policy, state Active) |
| Rule | `IT-HR-Conflict` |
| Left roles | `IT-Dept-Access` |
| Conflicting roles | `HR-Speciality-Access` |
| Violation owner | `spadmin` |

Export: [`exports/IT-vs-HR-SOD.xml`](../exports/IT-vs-HR-SOD.xml).

## 11. Outlier analysis

This 8.1 install has no dedicated outlier-detection task, so the analysis used **Intelligence → Advanced Analytics**:

1. Identity search with Department = `IT` → four identities.
2. **Result Options → Show Entitlements** → peer-group breakdown. `Department = IT` is held by 100% of the group; each job title by 25%.
3. Identity search with JobTitle = `HR Specialist` → only Alice Smith, who is in IT.

## 12. Lessons learned

- Delimited-file applications cannot be written to, so provisioning always produces manual work items.
- Load test data in its starting state, because attribute-change events need a change to fire.
- Check role assignment rules against leaver status, not only department.
- Keep an authoritative source and an entitlement source separate, even when both read similar data.
