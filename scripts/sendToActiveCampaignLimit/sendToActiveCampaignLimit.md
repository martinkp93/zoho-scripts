---
Function ID: "157805000001381060"
Name: validation_rule.sendToActiveCampaignLimit
Revision Timestamp: 2026-03-20T13:48:48.743Z
Status: Functional
---
**Postman Documentation:** [Link to API Collection Placeholder]

---

## Overview
The `validation_rule.sendToActiveCampaignLimit` function is a Zoho CRM Validation Rule script. Its purpose is to prevent a Sales Sprint from being activated (or sent to Active Campaign) if any of its associated Distributors (Accounts) are already participating in another active Sales Sprint. This prevents a single distributor from receiving conflicting campaign communications.

## Technical Contract
- **Input:** `String crmAPIRequest` (JSON payload from CRM Validation Rule).
- **Output:** `Map` (Containing `status` and `message` for the CRM UI).
- **Primary Entities:** 
    - `Sales_Sprints` (Current record context)
    - `Accounts` (Related Distributors)
    - `Related_Distributor_Accounts` (Linking module/Related List)

## Dependency Map
This script orchestrates the following internal functions and external services:

| Function / Service | Purpose | Criticality |
| --- | --- | --- |
| `Zoho CRM COQL API` | Retrieves all globally active Sales Sprints for comparison. | High |

## Logic Flow

```mermaid
graph TD
    Start(["Start: Validation Rule Trigger"]) --> Parse["Parse crmAPIRequest"]
    Parse --> GetDistributors["Get Related Distributors for this Sprint"]
    GetDistributors --> COQL["COQL Query: Find all Active Sprints (Global)"]
    COQL --> LoopDistributors["Loop: For Each Related Distributor"]
    LoopDistributors --> GetDistributorSprints["Get all Sprints for this Distributor"]
    GetDistributorSprints --> Intersect["Intersect Distributor Sprints with Global Active Sprints"]
    Intersect --> CheckConflict{"Conflict Found?"}
    CheckConflict -- "Yes" --> BuildConflictList["Add Sprint Name to alreadyActiveList"]
    CheckConflict -- "No" --> NextDistributor["Move to Next Distributor"]
    BuildConflictList --> NextDistributor
    NextDistributor --> FinalCheck{"Is alreadyActiveList empty?"}
    FinalCheck -- "No" --> Error["Return 'error' status with Conflict Names"]
    FinalCheck -- "Yes" --> Success["Return 'success' status"]
    Error --> End(["End"])
    Success --> End
```

## Core Logic Sections

### 1. Distributor Identification
The script identifies all distributors linked to the current Sales Sprint record via the `Related_Distributor_Accounts` related list.

### 2. Global State Comparison (COQL)
It performs a COQL query to identify every Sales Sprint in the system that is currently marked as `Active` and enabled for `Active Campaign`. These IDs are stored in a master list for comparison.

### 3. Nested Validation Loop
For every distributor linked to the current record, the script:
1. Retrieves all Sales Sprints associated with that specific distributor.
2. Uses the `.intersect()` method to see if any of those associated sprints match the global list of "Active" sprints.
3. Collects the names of conflicting sprints to provide a descriptive error message to the user.

## Developer Notes

> [!CAUTION]
> **Hardcoded ID Bug:** The current version of the script contains a hardcoded ID `recordId = 520877000208751093;`. This bypasses the dynamic `crmAPIRequest` and will cause the validation to fail or point to the wrong record in production. This must be changed to `recordId = recordMap.get("id");`.

> [!CAUTION]
> **Logic Scope Issue:** The variable `salesSprintIntersect` is calculated inside the distributor loop, but the logic to populate `alreadyActiveList` is currently placed *outside* the loop in the provided snippet. This means the script currently only validates conflicts for the *last* distributor in the list. The `alreadyActiveList` population logic should be moved inside the distributor loop.

> [!TIP]
> This script returns a specific Map format `{"status": "error", "message": "..."}` which is required for Zoho CRM Validation Rules to display custom error messages directly on the record UI.

## Change Log
- **2026-03-20T12:22:15.384Z:** Initial creation of documentation. Logic identified as a validation rule for distributor-campaign constraints.
- **2026-03-20T13:48:48.743Z:** Updated script to handle multiple distributors per Sales Sprint. Switched from single-account validation to a nested loop checking all linked distributors. Refined error message to return names of conflicting sprints. Added warnings regarding hardcoded IDs and loop scoping.