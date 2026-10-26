# Submit Button (OnSelect)

Validates the form, finds the approvers, creates the request and
sends the first approval mail.

```
Set(varApprover,
    If(!IsBlank(drpPlant.Selected.Value) && !IsBlank(drpCommodity.Selected.Value),
        LookUp('PAS-Approver Master',
            Plant.Value = drpPlant.Selected.Value &&
            'Commodity.'.Value = drpCommodity.Selected.Value
        )
    )
);
If(
    IsBlank(drpPlant.Selected.Value) ||
    IsBlank(drpCommodity.Selected.Value) ||
    IsBlank(drpSubCommodity.Selected.Value) ||
    IsBlank(drpContract.Selected.Value) ||
    IsBlank(DataCardValue3.Selected.Value) ||
    IsEmpty(FileUpload.Attachments) ||
    (drpSubCommodity.Selected.Value = "Others" && IsBlank(DataCardValue4.Text)),

    Notify("Please fill all required fields", NotificationType.Error),

    If(IsBlank(varApprover),
        Notify("No approver found for this Plant/Commodity combination", NotificationType.Error),
        Set(varIsExempt, drpContract.Selected.Value = "Contractual" && !IsBlank(varApprover.'L0 Approver'));
        Set(varFirst, If(varIsExempt, varApprover.'L0 Approver', varApprover.'L1 Approver'));
        Set(varNewRequest,
            Patch('PAS-Price Tracker', Defaults('PAS-Price Tracker'), EntryForm.Updates,
                {
                    'Request ID': varRequestID,
                    Date: Now(),
                    Requestor: User().FullName,
                    'Requestor Email': User().Email,
                    'Current Approval Level': {Value: If(varIsExempt, "0", "1")},
                    'Is Exemption': {Value: If(varIsExempt, "Yes", "No")},
                    'L0 Approver': If(varIsExempt, varApprover.'L0 Approver'.DisplayName),
                    'L0 Email': If(varIsExempt, varApprover.'L0 Approver'.Email),
                    Status: {Value: "Open"},
                    'ROB Category': If(drpSubCommodity.Selected.Value = "Others", DataCardValue4.Text, Blank()),
                    'Contract Type': {Value: drpContract.Selected.Value},
                    'SAP Upload Required': {Value: DataCardValue3.Selected.Value},
                    'SAP Uploader Name': If(DataCardValue3.Selected.Value = "Yes", varApprover.'SAP Upload'.DisplayName, Blank()),
                    'SAP Uploader Email': If(DataCardValue3.Selected.Value = "Yes", varApprover.'SAP Upload'.Email, Blank()),
                    'Level 1 Approver': varApprover.'L1 Approver'.DisplayName,
                    'L1 Email': varApprover.'L1 Approver'.Email,
                    'Level 2 Approver': If(drpContract.Selected.Value = "Non- Contractual", varApprover.'L2 Approver'.DisplayName),
                    'L2 Email': If(drpContract.Selected.Value = "Non- Contractual", varApprover.'L2 Approver'.Email),
                    'Level 3 Approver': If(drpContract.Selected.Value = "Non- Contractual", varApprover.'L3 Approver'.DisplayName),
                    'L3 Email': If(drpContract.Selected.Value = "Non- Contractual", varApprover.'L3 Approver'.Email)
                }
            )
        );

        Office365Outlook.SendEmailV2(
            varFirst.Email,
            "Price Change Approval Required - " & varNewRequest.'Request ID',
            "Hi " & varFirst.DisplayName & ",<br><br>" &
            "A new Price Change Approval request from " & User().FullName & " requires your approval.<br><br>" &
            "Request ID: " & varNewRequest.'Request ID' & "<br>" &
            "Plant: " & drpPlant.Selected.Value & "<br>" &
            "Commodity: " & drpCommodity.Selected.Value & "<br>" &
            "Sub-Commodity: " & drpSubCommodity.Selected.Value & "<br>" &
            If(drpSubCommodity.Selected.Value = "Others", "ROB Type: " & DataCardValue4.Text & "<br>", "") &
            "Contract Type: " & drpContract.Selected.Value & "<br>" &
            "SAP Upload: " & DataCardValue3.Selected.Value & "<br>" &
            "Remarks: " & Coalesce(DataCardValue5.Text, "-") & "<br><br>" &
            "Please review the attached file and respond via the approval screen:<br>" &
            "<a href=""<APP_LINK>"">Click here to review</a>",
            {
                IsHtml: true,
                Attachments: ForAll(
                    FileUpload.Attachments,
                    {
                        '@odata.type': "#Microsoft.OutlookServices.FileAttachment",
                        Name: ThisRecord.Name,
                        ContentBytes: ThisRecord.Value
                    }
                )
            }
        );

        Notify("Request Submitted Successfully and your Request ID is " & varRequestID, NotificationType.Success);
        ResetForm(EntryForm);
        Navigate(scrHome)
    )
)
```

## How it works
1. **Find the approvers:** Once Plant and Commodity are selected, the
   app looks up the matching row in the Approver Master list (the mail
   matrix) and stores it in `varApprover`.

2. **Validate the form:** Plant, Commodity, Sub Commodity, Contract
   Type, SAP Upload and the attachment are required. "Others"
   sub-commodity also needs a description. If anything is missing, an
   error is shown and nothing is saved.

3. **Check the approver exists:** If the matrix has no row for that
   Plant and Commodity, the request is stopped with an error.

4. **Decide the route:** A Contractual request is treated as an
   exemption if the matrix has an L0 approver for that Plant and
   Commodity, and starts at L0. All other requests start at L1.

5. **Create the request:** The request is saved to the tracker list
   with status Open, plus:
   - Requestor details and the starting approval level
   - Contract Type and SAP upload details
   - L1 approver details always, and L2 and L3 only for
     Non-Contractual requests

6. **Notify the first approver:** An email goes to the first approver
   with a request summary, the uploaded file attached and a link to the
   approval screen.

7. **Finish:** A success message shows the Request ID, the form is
   reset and the user returns to the home screen.
