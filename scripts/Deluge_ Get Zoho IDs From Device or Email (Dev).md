---
title: "Get Zoho IDs From Device or Email (Dev)"
module: standalone
type: script
status: active
---

# Deluge: Get Zoho IDs From Device or Email (Dev)

## Description
This script retrieves a comprehensive set of Zoho Desk and Zoho CRM identifiers, account details, and distributor information for a user. It accepts either a `device_label` (which triggers a lookup chain through Tycho and Populace) or a direct `email` address. The script is specifically configured for development/test environments, filtering Desk contacts with a `dev:true` custom field.

## Dependency Map
### Dependencies
*   None

### Functions Called
*   None

## Logic Flow

```mermaid
graph TD
    A[Start: crmAPIRequest] --> B[Parse request_body]
    B --> C{Validation: Email OR Device?}
    C -- No or Both --> D[Return 400/404 Error]
    
    C -- Device Label --> E[Invoke Tycho: Get Workspace ID]
    E --> F{Status 200?}
    F -- No --> G[Return 404 Error]
    F -- Yes --> H[Invoke Populace: Get User ID]
    H --> I{Status 200?}
    I -- No --> J[Return 404 Error]
    I -- Yes --> K[Desk: Search Contact by User ID & dev:true]
    
    C -- Email --> L[Desk: Search Contact by Email & dev:true]
    
    K --> M{Contact Count == 1?}
    L --> M
    M -- 0 Found --> N[Return 404 Error]
    M -- >1 Found --> O[Return 409 Error]
    
    M -- 1 Found --> P[Fetch Desk Account details]
    P --> Q[Fetch CRM Account details]
    Q --> R{Distributor Assigned?}
    
    R -- Yes --> S[Get CRM Distributor Account]
    S --> T[Fetch Desk Secondary Contacts for Account]
    T --> U[Identify Distributor Contact]
    U --> V[Build Success Response]
    
    R -- No --> V
    V --> W[Return 200 crmAPIResponse]
```

## Developer Notes
- [!IMPORTANT]
  The script enforces a strict "one or the other" rule for inputs. Providing both `email` and `device_label` or providing neither will result in a 404 validation error.
- [!NOTE]
  This script uses three distinct external connections: `tychodev`, `populacedev`, and `zohooauth`.
- [!SUCCESS]
  The script filters Zoho Desk contacts using `customField2=dev:true`. This ensures that production data is not inadvertently targeted during development testing.
- The distributor lookup logic specifically searches for a contact with the mapping type `SECONDARY` within the Desk Account to identify the distributor's representative.
- The CRM Account lookup utilizes the `Kanisa_Farm_ID` field as the Workspace ID reference.

## Change Log
| Timestamp | Change Description |
| :--- | :--- |
| 2026-03-19T14:09:33.993Z | Initial documentation of the development script. Implemented multi-service lookup (Tycho/Populace/Desk/CRM) to resolve IDs and distributor relationships. |