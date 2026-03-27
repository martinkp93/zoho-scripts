---
Function ID: "157805000001112005"
Name: delugeSyncActiveCampaignContact
Revision Timestamp: 2026-03-27T13:29:08.120Z
Status: Functional
---
**Postman Documentation:** [Link to API Collection Placeholder]

---

## Overview
The `delugeSyncActiveCampaignContact` function serves as an orchestration layer within the Cordulus ecosystem to ensure Zoho CRM Contacts are synchronized with ActiveCampaign. It specifically targets records that lack an `activeCampaignContactId`, triggers the synchronization handler, and updates the CRM record with the resulting external ID to maintain data parity.

## Technical Contract
- **Input:** 
    - `contactId` (Int): The Zoho CRM Contact record ID.
    - `accountId` (Int): The associated Zoho CRM Account record ID.
    - `firstName` (String): Contact's first name.
    - `lastName` (String): Contact's last name.
    - `email` (String): Contact's email address.
    - `country` (String): Contact's country.
    - `activeCampaignContactId` (String): The existing ID from ActiveCampaign (if any).
- **Output:** `void` (Side effect: Updates Zoho CRM record).
- **Primary Entities:** 
    - Zoho CRM `Contacts` Module.
    - ActiveCampaign Contact records.

## Dependency Map
This script orchestrates the following internal functions and external services:

| Function / Service | Purpose | Criticality |
| --- | --- | --- |
| [[delugeActiveCampaignHandler]] | Manages the direct API integration logic and payload formatting for ActiveCampaign. | High |
| Zoho CRM API | Used to update the contact record with the synced ID. | High |

## Logic Flow

```mermaid
graph TD
    Start(["Start"]) --> CheckID{"Is AC Contact ID Empty?"}
    CheckID -- "No" --> End(["End"])
    CheckID -- "Yes" --> SetTags["Set Default Tags: Customer, Employee"]
    SetTags --> BuildMap["Build Payload Map"]
    BuildMap --> CallHandler["Call [[delugeActiveCampaignHandler]]"]
    CallHandler --> CheckResp{"Did Handler Return ID?"}
    CheckResp -- "No" --> End
    CheckResp -- "Yes" --> UpdateCRM["Update CRM Contact: ActiveCampaign_Contact_Id"]
    UpdateCRM --> End
```

## Core Logic Sections

### 1. Synchronization Trigger Condition
The script first evaluates the `activeCampaignContactId` parameter. It only proceeds with the sync logic if this value is empty, preventing redundant API calls for contacts already mapped to ActiveCampaign.

### 2. Standalone Handler Invocation
The script packages contact details and the static tags (`{"Customer", "Employee"}`) into a Map called `payload`. It then passes this single Map to `[[delugeActiveCampaignHandler]]`. This refactor moves away from positional arguments to a more robust key-value structure.

> [!TIP]
> Using a Map for the handler payload allows for easier future expansion (e.g., adding more contact fields) without breaking the function signature of the handler.

### 3. Record Persistence
Upon a successful response from the handler, the script extracts the `acContactId`. It then performs a `zoho.crm.updateRecord` call to write this ID back to the `ActiveCampaign_Contact_Id` field on the Zoho CRM Contact record.

## Developer Notes

> [!WARNING]
> **Data Casting:** The script explicitly converts `contactId` and `accountId` to Long using `.toLong()` when building the payload map. Ensure the input `Int` does not exceed standard integer limits.

> [!IMPORTANT]
> **Response Handling:** The script checks if `syncAcContactResp.get("acContactId") != null`. If the handler returns an error map without this key, the CRM update will be skipped silently. Monitoring the `info` logs is required for debugging failed syncs.

> [!CAUTION]
> **Variable Scope:** The variable `syncAcContactResp` is initialized inside the first `if` block. If `activeCampaignContactId` is NOT empty, the second `if` statement attempts to call `.get()` on a potentially uninitialized variable. While Deluge sometimes defaults null/empty in these cases, it is best practice to initialize the map at the top of the script.

## Change Log
- **2026-03-19T19:33:50.220Z:** Initial creation of documentation via DeluluDocu.
- **2026-03-27T13:29:08.120Z:** Refactored the integration logic to use a Map-based payload when calling `[[delugeActiveCampaignHandler]]` instead of multiple positional arguments. This improves code readability and reduces errors related to argument ordering.