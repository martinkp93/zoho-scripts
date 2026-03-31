---
Function ID: "157805000000988006"
Name: delugeMobileAppRequestAQuote
Revision Timestamp: 2026-03-31T09:27:32.895Z
Status: Functional
---
**Postman Documentation:** [Link to API Collection Placeholder]

---

## Overview
The `delugeMobileAppRequestAQuote` function serves as an API entry point for the Cordulus Farm mobile application. When a user requests a quote via the app, this script synchronizes the selected distributor to the user's CRM Account, creates a tracking record in the "Conversions" module (including UTM metadata), and triggers a marketing automation update in ActiveCampaign via a secondary handler using a structured data payload.

## Technical Contract
- **Input:** `String crmAPIRequest` (A JSON-formatted string containing `workspaceId`, `userId`, and `distributorId`).
- **Output:** `Map` (containing `crmAPIResponse` with HTTP status codes 200 or 404 and a corresponding JSON body).
- **Primary Entities:** 
    - **Accounts:** Queried by `Kanisa_Farm_ID` and `Populace_Distributor_ID`.
    - **Contacts:** Queried by `Kanisa_User_ID`.
    - **Conversions:** A custom module record created to track "Request a Quote" events, now tracking both UTM Source and UTM Traffic.
    - **ActiveCampaign:** Updated via internal script call.

## Dependency Map
This script orchestrates the following internal functions and external services:

| Function / Service | Purpose | Criticality |
| --- | --- | --- |
| [[delugeActiveCampaignHandler]] | Synchronizes the conversion event and tags to the ActiveCampaign platform using a Map payload. | High |
| Zoho CRM (Search/Update/Create) | Standard CRUD operations for Accounts, Contacts, and Conversions. | Critical |

## Logic Flow

```mermaid
graph TD
    Start(["API Request Received"]) --> Parse["Parse JSON Body (workspaceId, userId, distributorId)"]
    Parse --> FindAcc{"Find CRM Account (workspaceId)"}
    FindAcc -- "Not Found" --> Err404_1["Return 404: Account Not Found"]
    FindAcc -- "Found" --> FindDist{"Find CRM Distributor (distributorId)"}
    FindDist -- "Not Found" --> Err404_2["Return 404: Distributor Not Found"]
    FindDist -- "Found" --> CompDist{"Selected == Current?"}
    CompDist -- "No" --> UpdateAcc["Update Account Distributor Field"]
    CompDist -- "Yes" --> FindCont{"Find CRM Contact (userId)"}
    UpdateAcc --> FindCont
    FindCont -- "Not Found" --> Err404_3["Return 404: Contact Not Found"]
    FindCont -- "Found" --> CheckAC{"Has ActiveCampaign ID?"}
    CheckAC -- "No" --> Err404_4["Return 404: Missing AC ID"]
    CheckAC -- "Yes" --> CreateConv["Create 'Conversions' Record with UTM Source/Traffic"]
    CreateConv --> CallAC["Invoke [[delugeActiveCampaignHandler]] with Map Payload (including Names)"]
    CallAC --> Success(["Return 200 Success"])
```

## Core Logic Sections

### 1. Entity Matching & Validation
The script performs a series of lookups to translate external Mobile App IDs into Zoho CRM internal IDs. It validates the existence of the Account (via `workspaceId`), the Distributor (via `distributorId`), and the Contact (via `userId`).

### 2. Distributor Synchronization
If the distributor selected in the mobile app does not match the distributor currently associated with the CRM Account, the script updates the Account's `Distributor_Lookup` field to maintain data consistency between the app and the CRM.

### 3. Conversion Attribution
A new record is created in the **Conversions** module with the following attributes:
- **Name:** "Request a Quote Conversion"
- **UTM_Source:** "Mobile App"
- **UTM_Traffic:** "Organic"

### 4. ActiveCampaign Integration
The script constructs a `payload` Map containing contact details (including `firstName` and `lastName` retrieved from the CRM), tags, and conversion metadata. This Map is passed to [[delugeActiveCampaignHandler]]. Passing a single Map object instead of individual parameters improves script maintainability and ensures all relevant contact identity data is synchronized.

## Developer Notes

> [!CAUTION]
> The script returns a `404` error if the Contact does not have an `ActiveCampaign_Contact_ID`. If ActiveCampaign synchronization is delayed or failed during contact creation, this mobile app request will fail.

> [!IMPORTANT]
> The call to [[delugeActiveCampaignHandler]] has been refactored. It no longer accepts positional arguments; it now requires a single `Map` containing the keys `contactId`, `accountId`, `email`, `tags`, etc.

> [!TIP]
> The script now extracts `First_Name` and `Last_Name` from the CRM Contact and includes them in the ActiveCampaign payload. This ensures that the marketing automation platform has the most up-to-date name information when processing the quote request.

## Change Log
- **2026-03-19T16:07:07.587Z:** Initial creation of documentation via DeluluDocu.
- **2026-03-27T13:29:10.785Z:** Updated script to refactor the ActiveCampaign handler call to use a Map-based payload. Added `UTM_Traffic` ("Organic") tracking to the CRM Conversion record creation and initialized variables at the top of the script for better readability.
- **2026-03-31T09:27:32.895Z:** Refactored ActiveCampaign integration to include Contact `firstName` and `lastName` in the payload passed to [[delugeActiveCampaignHandler]]. This ensures name data consistency between Zoho CRM and ActiveCampaign during the quote request process.