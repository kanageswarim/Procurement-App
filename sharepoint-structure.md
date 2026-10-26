# SharePoint Structure

Column names only. No data is included.

## Request Tracker
Stores every request and its approval progress.

| Group | Columns |
|---|---|
| Request details | Request ID, Date, Plant, Commodity, Sub Commodity, Others Category, Contract Type, Remarks, Attachment |
| Control | Status, Exemption Case, Current Approval Level |
| SAP upload | SAP Upload Required, SAP Uploader Name, Upload Status, Processed, SAP Email Sent |
| Approval levels (L0, L1, L2, L3) | Approver, Email, Status, Remarks, Date (repeated for each level) |

## Price Change Tracker
Stores the price details of approved requests.

Request ID, Date, Plant, Commodity, Sub Commodity, Contract Type, Part No, Supplier, New Quota %, New Price, New Unit Price, Reason for Price Change, New Price Valid from, Savings Type

## Downloads
Plant, Attachment

## Approver Master (Mail Matrix)
Maps each Plant and Commodity to the people who approve and are notified.
It drives approval routing and every mail sent.

Plant, Commodity, L0 Approver, L1 Approver, L2 Approver, L3 Approver, SAP Designated Person, Controller
