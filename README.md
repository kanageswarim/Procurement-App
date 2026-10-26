# Procurement Approval System

A multi-level approval system for procurement price-change and purchase
requests, built with Power Apps, SharePoint and Power Automate.

> Built at work for a procurement team.
> This repository contains no company data. Names, plants, emails and
> values are generic or dummy.

## Problem
Price-change approvals were handled over email and Excel, with no
tracking, no fixed approval chain and no record of who approved what.

## Solution
- Request submission with an attached Excel file
- Automatic routing by contract type (Contractual / Non-Contractual)
- Plant Level Exemption path
- Approval and rejection mails at each stage
- Request tracker, plus SAP upload support after final approval

## Architecture

```mermaid
flowchart LR
U[Requestor] --> PA[Power Apps<br>Request, Approval, Tracker]
PA <--> SP[(SharePoint<br>Request data, Attachments,<br>Downloads, Mail matrix)]
SP -->|Status = Approved| PF[Power Automate<br>Attachment to SharePoint]
PF --> SP
SP -->|SAP upload required| PF2[Power Automate<br>SAP upload files]
PF2 --> M[Email to SAP Designated Person]
PA --> E[Approval and info mails]
```

## Tech Stack
| Tool | Role |
|---|---|
| Power Apps | Request Submission, Approval screens, Request Tracker, SAP Upload confirmation; routing, status updates and mails coded in the app |
| SharePoint | Request data, Attachment content, Downloads, Mail matrix |
| Power Automate | Moves approved attachment content into SharePoint; builds SAP Upload files and emails them |

## Approval Flows

### Non-Contractual (3 levels)
```mermaid
flowchart TD
A([Request Submitted]) --> B[L1 Approval<br>Cluster Head]
B -->|Rejected| R1[Info mail: Requestor]
B -->|Approved| C[L2 Approval<br>Plant HOP]
C -->|Rejected| R2[Info mail: Requestor + L1]
C -->|Approved| D[L3 Approval<br>Procurement Head]
D -->|Rejected| R3[Info mail: Requestor + L1 + L2]
D -->|Approved| E[Info mail: Requestor]
E --> F{SAP Upload required?}
F -->|Yes| G[Mail: SAP Designated Person]
F -->|No| H[Info mail: Controller]
G --> H
H --> Z([End])
R1 --> Z
R2 --> Z
R3 --> Z
```

### Contractual (1 level, with Plant Level Exemption)
```mermaid
flowchart TD
A([Request Submitted]) --> B{Normal or<br>Plant Level Exemption?}
B -->|Normal| C[L1 Approval<br>Cluster Head]
B -->|Plant Level Exemption| D[L0 Approval<br>Local Cluster Head]
C -->|Rejected| R1[Info mail: Requestor]
C -->|Approved| A1[Info mail: Requestor + L2]
D -->|Rejected| R2[Info mail: Requestor]
D -->|Approved| A2[Info mail: Requestor + L1 + L2]
A1 --> F{SAP Upload required?}
A2 --> F
F -->|Yes| G[Mail: SAP Designated Person]
F -->|No| H[Info mail: Controller]
G --> H
H --> Z([End])
R1 --> Z
R2 --> Z
```

The exemption path is chosen automatically from pre-configured
plant and commodity combinations.


## Code Snippets
Sanitised Power Apps logic, with links, IDs and names removed:

- [Submit button](code-snippets/submit-button.md): validation, approver
  lookup, request creation and the first approval mail
- [Approve button (L0/L1)](code-snippets/approve-button-l0-l1.md):
  rejection, final approval for Contractual requests, and routing
  Non-Contractual requests to the next level

The L2 and L3 approval buttons follow the same pattern as the L0/L1
button: they update that level's status, comments and date, then either
notify the next approver or send the final approval mail.

For More Details:
- [Power Automate flows](power-automate-flows.md)
- [SharePoint structure](sharepoint-structure.md)
