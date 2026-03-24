---
Function ID: "157805000001393007"
Name: delugeSendToActiveCampaignLimit
Revision Timestamp: 2026-03-24T14:32:42.641Z
Status: Functional
---

**Postman Documentation:** [Link to API Collection Placeholder]

---

## Overview
The `delugeSendToActiveCampaignLimit` function is a validation utility designed to identify intersections between specific Accounts (distributors) and currently active Sales Sprints. It processes a payload of names, resolves them to CRM IDs, fetches globally active Sales Sprints via COQL, and determines which distributors are associated with those active sprints. The latest update introduces a specific filter for "Farm_Distributor" during the initial Account resolution phase.

## Technical Contract
- **Input:** `String payload` (Expected to be an iterable collection of Account names).
- **Output:** `String` (A stringified Map containing `status` and `message` if conflicts exist, or the COQL response object).
- **Primary Entities:** 
    - Zoho CRM (Accounts Module)
    - Zoho CRM (Sales_Sprints Module)
    - COQL (CRM Object Query Language)
    - ActiveCampaign (Contextual destination)

## Dependency Map
This script orchestrates the following internal functions and external services:

| Function / Service | Purpose | Criticality |
| --- | --- | --- |
| Zoho CRM (Accounts) | Searches for account records based on names and `Distributor_Type`. | High |
| Zoho CRM (COQL) | Fetches active Sales Sprints via `https://www.zohoapis.eu/crm/v2/coql`. | High |
| Connection: `zohocrmconnection` | OAuth2 connection for COQL API execution. | High |
| Zoho CRM (Related Records) | Retrieves "Related_Sales_Sprints_2" for specific Accounts. | High |

## Logic Flow
The function resolves Account names to IDs (filtering for Farm Distributors), queries the CRM for globally active Sales Sprints, and iterates through distributors to find overlaps. It then validates the results to generate a descriptive conflict message.

```mermaid
graph TD
    Start["Start: Receive Payload"] --> ResolveDistributors["Iterate Payload: Resolve Account Names (Type: Farm_Distributor)"]
    ResolveDistributors --> COQLQuery["Execute COQL: Get Global Active Sales Sprints"]
    COQLQuery --> MapActiveIDs["Map Active Sprint IDs to List"]
    MapActiveIDs --> DistLoopStart{"For each resolved Distributor ID"}
    DistLoopStart -- "Process" --> GetRelated["zoho.crm.getRelatedRecords('Related_Sales_Sprints_2')"]
    GetRelated --> Intersect["Calculate Intersection (Active vs Related)"]
    Intersect --> CheckSize{"Intersection > 0?"}
    CheckSize -- "Yes" --> AddToList["Add to intersectList"]
    CheckSize -- "No" --> DistLoopStart
    AddToList --> DistLoopStart
    DistLoopStart -- "Finished" --> CleanList["Clean List (Distinct & Remove Nulls)"]
    CleanList --> ConflictCheck{"Conflicts Found?"}
    ConflictCheck -- "Yes" --> InitErrorMap["Initialize response = Map()"]
    InitErrorMap --> BuildConflictMsg["Loop through Conflicts: Build Error Message"]
    ConflictCheck -- "No" --> ReturnResponse["Return Response (COQL Search Map)"]
    BuildConflictMsg --> ReturnResponse
    ReturnResponse --> End["End"]
```

## Core Logic Sections
The script consists of the following logical components:

### 1. Distributor Resolution (Updated)
The script loops through the input `payload`, performing a `zoho.crm.searchRecords` for each name. The search criteria now explicitly filters for `Distributor_Type:equals:Farm_Distributor`. Results are stored in a temporary `search` variable.

### 2. Global Active Sprint Discovery (COQL)
Utilizes a COQL query to target the `Sales_Sprints` module, filtering for records where `Sales_Sprint_Active` is 'Yes' and `Send_to_Active_Campaign` is true.

### 3. Relationship Intersection Logic
For every resolved distributor, the script:
1.  Fetches their specific related sprints via `getRelatedRecords`.
2.  Uses the `.intersect()` method to find common IDs between the global "Active" list and the distributor's "Related" list.

### 4. Validation and Conflict Reporting
The script evaluates the `intersectList`. If conflicts exist:
1.  The `response` variable is explicitly initialized as a new `Map()`.
2.  It iterates through the unique conflict IDs to build a detailed string listing the conflicting entities.
3.  The map is populated with an "error" status and the message.

## Developer Notes

> [!CAUTION]
> **Syntax Warning:** In the latest update, the search criteria string `"(Account_Name:equals:" + name +") AND (Distributor_Type:equals:Farm_Distributor"` appears to be missing a closing parenthesis at the end of the criteria. This may cause the search to fail or throw an exception in some CRM environments.

> [!IMPORTANT]
> This script uses an `invokeurl` call for COQL using the `zohocrmconnection`. Ensure this connection exists in the Zoho environment with the `ZohoCRM.coql.READ` scope.

> [!CAUTION]
> **Logic Limitation:** The validation loop for `conflictMessage` relies on the `relatedSalesSprints` variable. However, this variable is updated inside the distributor loop. Consequently, the final validation logic only has access to the related records of the *last* distributor processed in the payload. If conflicts occur across multiple different distributors, the name mapping in the error message may be incomplete or incorrect.

> [!CAUTION]
> **Performance Warning:** The script performs a `getRelatedRecords` call *inside* a loop for every distributor in the payload. If the payload contains many names, this will execute multiple additional API calls, potentially hitting CRM limits.

> [!TIP]
> **Variable Scoping Fix:** The previous update renamed intermediate API response variables to `search`. This prevents the script from accidentally returning partial API data or trying to `put` error keys into a non-map object if the COQL call failed or returned unexpected types.

## Change Log
- **2026-03-24T13:44:57.179Z:** Initial creation of documentation via DeluluDocu.
- **2026-03-24T14:16:16.993Z:** Updated script logic to include a `for each` loop and `zoho.crm.searchRecords` integration.
- **2026-03-24T14:17:58.763Z:** Updated logic to extract the specific CRM Record ID (`distributorId`) from the search response.
- **2026-03-24T14:18:26.815Z:** Corrected index handling for CRM search results using `.get(0)`.
- **2026-03-24T14:22:02.736Z:** Major update: Integrated COQL query to fetch active Sales Sprints and implemented intersection logic to validate distributors against active sprints. Added `invokeurl` dependency and related record processing.
- **2026-03-24T14:23:01.022Z:** Maintenance: Commented out several `info` debug statements throughout the script to clean up execution output while preserving core logic.
- **2026-03-24T14:27:12.400Z:** Added Part B: Validation Logic. The script now performs a distinct check on conflicts and returns a structured "error" status map with a descriptive `conflictMessage` containing Distributor names and IDs if intersections are found.
- **2026-03-24T14:28:35.469Z:** Bug fix and Refactor: Renamed intermediate API result variables from `response` to `search` to prevent variable collision. Explicitly initialized `response = Map()` within the validation block to ensure a clean error object is returned when conflicts are detected.
- **2026-03-24T14:32:42.641Z:** Search Filter Refinement: Updated the `zoho.crm.searchRecords` call for Accounts to include a mandatory check for `Distributor_Type:equals:Farm_Distributor`. Commented out remaining `info` debug statements.