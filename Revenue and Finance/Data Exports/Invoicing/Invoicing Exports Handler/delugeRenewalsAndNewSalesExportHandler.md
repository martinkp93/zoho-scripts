---
Function ID: "157805000001307001"
Name: delugeRenewalsAndNewSalesExportHandler
Revision Timestamp: 2026-03-30T04:54:50.981Z
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
    - **Zoho CRM**: Accounts (Distributors) and Settings Variables (Job IDs).
    - **Zoho Analytics**: Bulk Export API v2.
    - **Google Sheets**: Individual Distributor spreadsheets and Master Dashboard (`1iM5nTGy...` or `15NpCTxm...`).
    - **Mailersend**: External email delivery service.

## Dependency Map
This script orchestrates the following internal functions and external services:

| Function / Service | Purpose | Criticality |
| --- | --- | --- |
| [[delugeSendErrorAlert]] | Handles error reporting to administrators if sheet creation or data clearing fails. | High |
| [[delugePostSuccessMessageToSlack]] | Sends a summary breakdown of processed quantities to the designated Slack channel. | Medium |
| **Zoho Analytics API** | Source of truth for the processed data records via Export Jobs. | Blockers |
| **Google Sheets API** | Target destination for data visualization and distribution. | Blockers |
| **Mailersend API** | Used to deliver XLSX exports to Cropline distributors. | Medium |

## Logic Flow

```mermaid
graph TD
    Start(["Start (Task, Segment)"]) --> FetchVars["Fetch Analytics JobID from CRM Variables"]
    FetchVars --> FetchCRM["Fetch Accounts with 'Renewals_and_New_Sales_Data' URL"]
    FetchCRM --> AnalyticsExport["Invoke Analytics Export Job Data"]
    AnalyticsExport --> FilterData["Filter Records by Segment/Allowed Products"]
    FilterData --> DateCalc["Calculate Target Month/Year for Tab Names"]
    DateCalc --> LoopDist["Loop each Distributor"]
    
    subgraph GoogleUpdate ["Google Sheets Operations"]
        LoopDist --> TabCheck{"Tab Exists?"}
        TabCheck -- "No" --> CreateTab["Create MMMM yyyy Tab"]
        TabCheck -- "Yes" --> ClearData["Clear Old Range A1:Z1000"]
        CreateTab --> ClearData
        ClearData --> PushData["Push Sorted Data & Summary Formulas"]
        PushData --> MasterDash["Update Master Dashboard Cell & Note"]
    end
    
    MasterDash --> IsCropline{"Is Segment Cropline?"}
    IsCropline -- "Yes" --> Email["Export XLSX & Send via Mailersend"]
    IsCropline -- "No" --> NextDist["Next Distributor"]
    Email --> NextDist
    
    NextDist --> End(["Post Slack Summary & End"])
```

## Core Logic Sections

### 1. Configuration & Data Retrieval
The script first identifies the `jobId` for the Analytics export by querying CRM Settings Variables. It then identifies all "Distributor" accounts in the CRM that have a spreadsheet URL configured. It dynamically selects either the standard or "Cropline" spreadsheet ID based on the `segment` input.

### 2. Zoho Analytics Integration
Using the Zoho Analytics REST API v2, the script fetches the JSON data resulting from a pre-configured Export Job. This job contains the calculated pricing, tiered discounts, and product details for the relevant period.

### 3. Data Transformation & Filtering
The data is filtered by `allowedProducts` (e.g., "Cordulus Farm" for the Cordulus segment). The script organizes the flat list of records into a Map where keys are Distributor names and values are lists of rows, including a standardized header row. 

### 4. Tab & Row Management
- **Target Dates**: If the task is "Renewals", it targets next month. If "New Sales", it targets the previous month.
- **Sorting**: Rows are sorted alphabetically by "Account Name" (Column C) using a temporary Map to ensure consistent presentation in the Google Sheet.
- **Formulas**: The script injects Google Sheets formulas (e.g., `=SUM(O8:O)`) at the top of the sheet for real-time summary calculations.

### 5. Master Dashboard Synchronization
The script locates the correct row in the "Master Tracking Dashboard" by matching the Distributor Name and Task. It calculates the total quantity of subscriptions/units and updates the specific cell corresponding to the target month. It also attaches a **Google Sheets Note** to the cell containing a timestamp of the update.

### 6. Notifications & Export
For the "Cropline" segment, the script generates an XLSX export of the newly updated tab and sends it via Mailersend to specified recipients. Finally, a summary of all processed quantities per distributor is posted to Slack.

## Developer Notes

> [!IMPORTANT]
> This script relies on exact matching of "Account Name" between Zoho CRM and the "Master Tracking Dashboard" spreadsheet. If a distributor's name is changed in CRM without updating the dashboard, the Master Dashboard update will fail (though the individual sheet will still update).

> [!WARNING]
> **Hardcoded IDs**: The script contains hardcoded Workspace IDs, Org IDs, and Dashboard Spreadsheet IDs. If the Analytics workspace or the Master spreadsheets are recreated, these variables must be updated manually.

> [!TIP]
> **Performance**: The script fetches the Master Dashboard row values once per execution (`dashboardMeta`) rather than per-distributor to avoid hitting Google Sheets API rate limits during large batch runs.

> [!NOTE]
> **Slack Notifications**: As of the latest update, the call to `[[delugePostSuccessMessageToSlack]]` is fully enabled, providing the operations team with a real-time summary of the quantities pushed to the sheets.

## Change Log
- **2026-03-19T19:39:37.540Z:** Initial creation of documentation via DeluluDocu. Added logic for dynamic Year/Month tab targeting and Mailersend integration for Cropline.
- **2026-03-19T20:30:02.120Z:** Re-validation of script logic. Confirmed identical operational logic for both "Renewals" and "New Sales" modes regarding Google Sheet clearing and dashboard note updates. Verified no functional changes in this revision code block.
- **2026-03-19T21:12:49.368Z:** Logic verification pass. Confirmed consistency across API endpoints (CRM v8 and Analytics v2). No functional code changes; updated documentation to reflect "commented out" status of Slack notifications and added standard developer notes for observability.
- **2026-03-30T04:54:50.981Z:** Enabled final Slack success notification. The `[[delugePostSuccessMessageToSlack]]` function is now active at the end of the script to provide a summary breakdown of quantities processed for the task and segment. No other functional changes were made to the core data processing or sheet update logic.