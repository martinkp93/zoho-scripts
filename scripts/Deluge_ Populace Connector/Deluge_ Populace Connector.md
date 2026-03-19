---
Function ID: "157805000001170034"
Name: Deluge: Populace Connector
Revision Timestamp: 2026-03-19T14:29:09.413Z
Status: Functional
---
**Postman Documentation:** [Link to API Collection Placeholder]

---

## Overview
The `delugePopulaceConnector` serves as a centralized API wrapper for interacting with the **Populace** service (an external system for managing users, workspaces, and distributors within the Cordulus ecosystem). 

It acts as a middleware layer that translates abstract actions (e.g., `createUser`) into specific HTTP methods and URL paths, handles dynamic path parameter substitution, executes the network request via the `populace` Zoho Connection, and provides standardized error reporting.

## Technical Contract
- **Input:** 
    - `String action`: A key corresponding to a predefined endpoint (e.g., `updateUser`, `addMembership`).
    - `Map payload`: Data required for the request. This map serves dual purposes: providing values for URL path parameters (like `{userId}`) and providing the JSON body for POST/PUT requests.
- **Output:** A Map containing:
    - `success`: (Boolean) Indicates if the request was successful (HTTP 200, 201, or 204).
    - `data`: (String) The raw response text from the API (on success).
    - `error_message`: (String) Descriptive error details (on failure).
- **Primary Entities:** 
    - Populace API (`https://populace.cordulus.com`)
    - Zoho Connection: `populace`

## Dependency Map
This script orchestrates the following internal functions and external services:

| Function / Service | Purpose | Criticality |
| --- | --- | --- |
| [[delugeSendErrorAlert]] | Logs failures and sends notifications when API calls fail or exceptions occur. | High |
| Populace API | External service for user and workspace orchestration. | Mission Critical |

## Logic Flow

```mermaid
graph TD
    Start([Start]) --> Config[Lookup Action in Config Map]
    Config --> CheckAction{Action Valid?}
    
    CheckAction -- No --> ErrAction[Call delugeSendErrorAlert]
    ErrAction --> RetFail[Return Success: False]
    
    CheckAction -- Yes --> PathParam[Iterate Payload Keys]
    PathParam --> Replace{Key in Path?}
    Replace -- Yes --> Sub[Replace {key} in Path & Remove from Payload]
    Replace -- No --> Next[Keep in Payload for Body]
    Sub --> PathParam
    Next --> PathParam
    
    PathParam -- Done --> Invoke{Method Type?}
    Invoke -- POST/PUT --> InvBody[InvokeURL with Payload as JSON Body]
    Invoke -- DELETE --> InvDel[InvokeURL without Body]
    
    InvBody --> RespCode{HTTP 200/201/204?}
    InvDel --> RespCode
    
    RespCode -- Yes --> RetSucc[Return Success: True + Data]
    RespCode -- No --> ErrAPI[Call delugeSendErrorAlert]
    ErrAPI --> RetFail
    
    catch[Exception Catch] --> ErrCatch[Call delugeSendErrorAlert]
    ErrCatch --> RetFail
```

## Core Logic Sections

### 1. API Endpoint Configuration
The script maintains a hardcoded dictionary (`config`) mapping logical actions to their respective HTTP methods and path templates. This centralizes the API definition, making it easier to add new endpoints without altering the execution logic.

### 2. Dynamic Path Interpolation
A robust loop iterates through the `payload`. If a key matches a placeholder in the URL (e.g., `{workspaceId}`), the script:
1. URL-encodes the value.
2. Replaces the placeholder in the path.
3. **Removes the key from the payload map.**
This ensures that the payload passed to the final `invokeurl` contains only the data intended for the request body, while path variables are handled separately.

### 3. Unified Execution & Error Handling
The script standardizes the `invokeurl` call using a named connection (`populace`). It implements a strict whitelist of success codes (200, 201, 204). Any other response code or script exception triggers a call to `[[delugeSendErrorAlert]]` and returns a structured error object.

## Developer Notes

> [!IMPORTANT]
> This script requires a Zoho Connection named `populace` to be pre-configured with the appropriate scopes for the external API.

> [!WARNING]
> The script modifies the `payload` map in-place during the path-substitution phase. If the calling script needs the original payload map after this function executes, it must pass a copy.

> [!CAUTION]
> If a path parameter is missing from the `payload` map but required by the URL template (e.g., `{userId}`), the literal string `{userId}` will remain in the URL, likely causing a 404 or 400 error from the API.

> [!SUCCESS]
> The interpolation logic handles complex paths with multiple parameters (e.g., `/distributors/{distributorId}/users/{userId}`) by iteratively updating the path and cleaning the payload body.

## Change Log
- **2026-03-19T14:28:08.009Z:** Initial creation of documentation via DeluluDocu.
- **2026-03-19T14:29:09.413Z:** Maintenance update; minor comment adjustment in Step 1 to clarify configuration maps. No logic changes were made to the core execution flow.