---
Function ID: "157805000001170034"
Name: Deluge: Populace Connector
Revision Timestamp: 2026-03-19T14:11:21.533Z
Status: Functional
---
**Postman Documentation:** [Link to API Collection Placeholder]

---

## Overview
The `delugePopulaceConnector` serves as the centralized wrapper for all interactions with the Cordulus Populace API. Its primary role is to abstract the complexities of URL construction, placeholder replacement for RESTful paths, and standardized error handling. By providing a single point of entry for operations related to Users, Workspaces, and Distributors, it ensures consistent API communication across the Cordulus Zoho environment.

## Technical Contract
- **Input:** 
    - `action` (String): The internal key representing the API operation (e.g., `createUser`, `addMembership`).
    - `payload` (Map): A data map containing both the body parameters and the values for URL path placeholders.
- **Output:** 
    - A Map containing `{"success": Boolean, "data": String/Map, "error_message": String}`.
- **Primary Entities:** 
    - Cordulus Populace API Service
    - Zoho CRM/Books (via calling scripts)

## Dependency Map
This script orchestrates the following internal functions and external services:

| Function / Service | Purpose | Criticality |
| --- | --- | --- |
| [[delugeSendErrorAlert]] | Dispatches notifications to developers when API calls fail or exceptions occur. | Medium |
| Populace Connection | Zoho OAuth connection used to authorize requests to `populace.cordulus.com`. | High |

## Logic Flow

```mermaid
graph TD
    Start([Start]) --> ActionMapping[Lookup Action in Config Map]
    ActionMapping --> ActionValid{Action Valid?}
    
    ActionValid -- No --> TriggerError[Call delugeSendErrorAlert]
    TriggerError --> ReturnFail[Return Success: False]
    
    ActionValid -- Yes --> PathParser[Loop Payload: Replace {key} in Path]
    PathParser --> FinalizeURL[Construct Full URL & Cleanse Payload Body]
    
    FinalizeURL --> MethodSwitch{HTTP Method?}
    MethodSwitch -- POST --> ExecPOST[Invoke POST]
    MethodSwitch -- PUT --> ExecPUT[Invoke PUT]
    MethodSwitch -- DELETE --> ExecDELETE[Invoke DELETE]
    
    ExecPOST --> ResponseEval
    ExecPUT --> ResponseEval
    ExecDELETE --> ResponseEval
    
    ResponseEval{Status Code 200/201/204?}
    ResponseEval -- Yes --> ReturnSuccess[Return Success: True + Data]
    ResponseEval -- No --> TriggerError
```

## Core Logic Sections

### 1. Action Configuration Map
The script initializes a comprehensive map that defines the HTTP method (`m`) and the URI path template (`p`) for every supported action. This makes the script easily extensible; adding a new endpoint only requires adding a line to this map.

### 2. RESTful Path Parsing
This section iterates through the `payload` keys. If a key matches a placeholder in the URI (e.g., `{userId}`), it encodes the value, injects it into the path, and **removes** that key from the payload. This ensures that URL parameters are not erroneously sent in the JSON body of the request.

### 3. Dynamic Invocation
Using the `invokeurl` task, the script executes the request against the `populace` connection. It supports `POST`, `PUT`, and `DELETE` methods, passing the cleansed payload as a stringified JSON body where applicable.

### 4. Standardized Response & Error Handling
The script evaluates the `responseCode`. It treats 200, 201, and 204 as successful operations. Any other status code or a script-level exception triggers the `delugeSendErrorAlert` function and returns a failure map to the calling script.

## Developer Notes

> [!WARNING]
> **Payload Mutation:** The script removes keys from the `payload` map once they are used as URL path parameters. If the API requires the same ID in both the URL and the JSON body, the calling script must provide a duplicate key or this script must be modified.

> [!IMPORTANT]
> **Connection Dependency:** This script requires a Zoho Northbound Connection named exactly `populace` with appropriate OAuth scopes to reach the Cordulus service.

> [!NOTE]
> All path parameters are automatically `encodeUrl()` to prevent breaks when IDs or names contain special characters.

## Change Log
- **2026-03-19T14:11:21.533Z:** Initial creation of documentation via DeluluDocu.