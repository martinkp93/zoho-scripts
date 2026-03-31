---
Function ID: "157805000001307001"
Name: delugeRenewalsAndNewSalesExportHandler
Revision Timestamp: 2026-03-31T21:45:05.250Z
Status: Functional
---
**Postman Documentation:** [Link to API Collection Placeholder]

---

## Overview
The `delugeRenewalsAndNewSalesExportHandler` script is a critical integration component that bridges Zoho Analytics, Zoho CRM, and Google Sheets. Its primary purpose is to take processed financial data (either upcoming Renewals or previous month's New Sales) exported from Zoho Analytics and distribute it into specific Google Spreadsheets owned by individual Distributors. 

Furthermore, it updates a centralized "Master Tracking Dashboard" for internal Cordulus monitoring and, for the "Cropline" segment, automatically exports the resulting data as an Excel file and emails it to stakeholders via Mailersend.

## Technical Contract
- **Input:** 
    - `String task`: Determines the logic mode. Accepted values: `"Renewals"` or `"New Sales"`.
    - `String segment`: Determines the product filtering and target spreadsheet columns. Accepted values: `"Cordulus"` or `"Cropline"`.
- **Output:** `String` (Returns an empty string upon completion).
- **Primary Entities:** 
    - **Zoho CRM**: Accounts (Distributors).
    - **Zoho Analytics**: REST API v2 (Views: 188580000008023658 / 188580000008023872).
    - **Google Sheets**: Individual Distributor spreadsheets and Master Dashboards (`1iM5nTGy...` or `15NpCTxm...`).
    - **Mailersend**: External email delivery service for Cropline exports.

## Dependency Map
This script orchestrates the following internal functions and external services:

| Function / Service | Purpose | Criticality |
| --- | --- | --- |
| [[delugeSendErrorAlert]] | Handles error reporting to administrators if sheet creation or data clearing fails. | High |
| [[delugePostSuccessMessageToSlack]] | (Optional) Sends a summary breakdown of processed quantities to Slack. | Medium |
| **Zoho Analytics API** | Source of truth for the processed data records via REST API v2 Export. | Blockers |
| **Google Sheets API** | Target destination for data visualization and master dashboard tracking. | Blockers |
| **Mailersend API** | Used to deliver XLSX exports to Cropline distributors. | Medium |

## Logic Flow

```mermaid
graph TD
    Start(["Start (Task, Segment)"]) --> FetchCRM["Fetch CRM Accounts with valid SS URLs"]
    FetchCRM --> AnalyticsExport["Invoke Analytics REST API v2 (Export View)"]
    AnalyticsExport --> FilterData["Filter Records by Segment & Product List"]
    FilterData --> DateCalc["Calculate Target Month/Year for Tabs"]
    DateCalc --> SortData["Sort Records Alphabetically by Account Name"]
    SortData --> LoopDist["Loop each Distributor with Data"]
    
    subgraph GoogleUpdate ["Google Sheets Operations"]
        LoopDist --> TabCheck{"Tab Exists in Target?"}
        TabCheck -- "No" --> CreateTab["Create MMMM yyyy Tab"]
        TabCheck -- "Yes" --> ClearData["Clear A1:Z1000"]
        CreateTab --> ClearData
        ClearData --> PushData["Push Sorted Rows & Summary Formulas"]
        PushData --> MasterDash["Update Master Dashboard Cell Value"]
        MasterDash --> CellNote["Add Timestamp Note to Master Cell"]
    end
    
    CellNote --> IsCropline{"Is Segment Cropline?"}
    IsCropline -- "Yes" --> Email["Export XLSX & Send via Mailersend"]
    IsCropline -- "No" --> NextDist["Next Distributor"]
    Email --> NextDist
    
    NextDist --> End(["Build Slack Summary & End"])
```

## Core Logic Sections

### 1. Configuration & CRM Data Fetching
The script searches CRM Accounts for distributors possessing a `Renewals_and_New_Sales_Data` link. It extracts the Spreadsheet ID from the URL and maps it to the distributor's ID and Name.

### 2. Zoho Analytics Data Retrieval
Using the Analytics REST API v2, the script pulls JSON records from specific views based on the `task`. 
- **Renewals**: View ID `188580000008023658`
- **New Sales**: View ID `188580000008023872`

### 3. Data Processing & Sorting
The data is filtered by `allowedProducts` (e.g., "Cordulus Farm" for Cordulus, "Cropline" for Cropline). Rows are organized into a Map by distributor. Before being pushed to Google Sheets, the rows are sorted alphabetically by "Account Name" (Column C). A unique suffix is added to keys during sorting to prevent data loss if multiple rows share the same account name.

### 4. Google Sheets Updates (Individual Tabs)
- **Tab Creation**: Checks for a tab named like "January 2026 (Renewals)". If missing, it creates it using `batchUpdate`.
- **Formulas**: Injects summary formulas at the top (Rows 2-3) to calculate totals for Quantity (`P`) and Price (`R`).
- **Cleaning**: Always clears `A1:Z1000` before pushing fresh data to ensure no ghost rows remain.

### 5. Master Dashboard Synchronization
The script performs a lookup on the Internal Tracking Dashboard spreadsheet.
- **Cell Selection**: Matches Distributor Name (Col A) and Task (Col B). The column is determined by a Month-to-Column map (e.g., May -> Col G).
- **Updates**: Pushes the `totalQty` of subscriptions (filtered by "Subscription" or "Cropline" keywords).
- **Notes**: Attaches a Google Sheets note to the updated cell with a detailed timestamp of the last update.

### 6. Cropline Emailing (Mailersend)
Specifically for the Cropline segment, the script exports the newly updated tab as an `.xlsx` file via the Google Sheets export endpoint, converts it to Base64, and sends it to a hardcoded recipient (`dela@danishagro.dk`) via the Mailersend API.

## Developer Notes

> [!IMPORTANT]
> **Account Name Matching**: The script relies on an exact string match for "Account Name" between Zoho CRM and the Master Tracking Dashboard. Any mismatch prevents the master dashboard cell from updating.

> [!WARNING]
> **Hardcoded Parameters**: The script contains hardcoded IDs for Analytics Workspaces, Google Dashboard IDs, and Slack Channel IDs. If these resources are moved or recreated, these constants must be updated.

> [!CAUTION]
> **Slack Notification Status**: In the current version of the code, the call to `[[delugePostSuccessMessageToSlack]]` is commented out. The logic to build the summary string exists, but the final delivery is disabled.

> [!TIP]
> **Handling Empty Values**: The script explicitly converts null or empty strings in the Analytics data to "0" to prevent potential calculation errors within the Google Sheets environment.

## Change Log
- **2026-03-19T19:39:37.540Z:** Initial creation of documentation via DeluluDocu. Added logic for dynamic Year/Month tab targeting and Mailersend integration for Cropline.
- **2026-03-19T20:30:02.120Z:** Re-validation of script logic. Confirmed identical operational logic for both "Renewals" and "New Sales" modes regarding Google Sheet clearing and dashboard note updates. Verified no functional changes in this revision code block.
- **2026-03-19T21:12:49.368Z:** Logic verification pass. Confirmed consistency across API endpoints (CRM v8 and Analytics v2). No functional code changes; updated documentation to reflect "commented out" status of Slack notifications and added standard developer notes for observability.
- **2026-03-30T04:54:50.981Z:** Enabled final Slack success notification. The `[[delugePostSuccessMessageToSlack]]` function is now active at the end of the script to provide a summary breakdown of quantities processed for the task and segment.
- **2026-03-31T13:21:50.259Z:** Adjusted Google Sheets summary formulas. Updated column indices from O and Q to **P** and **R** respectively to align with updated export data mapping. Confirmed that the `[[delugePostSuccessMessageToSlack]]` function remains active.
- **2026-03-31T21:45:05.250Z:** Updated documentation to reflect code-level changes where `[[delugePostSuccessMessageToSlack]]` was commented out in the latest script source. Verified internal sorting logic and the addition of automated Google Sheets cell notes with timestamps for auditability. Verified column indices for summary formulas (P8 and R8).