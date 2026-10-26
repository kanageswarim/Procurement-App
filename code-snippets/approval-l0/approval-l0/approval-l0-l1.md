# Approval Button (OnSelect)

```Set(varAttach, ForAll(First(Gallery2.AllItems).Attachments,
    {Name: ThisRecord.DisplayName, ContentBytes: JSON(ThisRecord.Value, JSONFormat.IncludeBinaryData)}));

If(
    IsBlank(Comments1.Text) || IsBlank(Radio1.Selected.Value),
    Notify("Please add comments and select Approved/Rejected", NotificationType.Error),

    With({varRecord: LookUp('PAS-Price Tracker', 'Request ID' = 'ID Label'.Text)},
    With({isEx: varRecord.'Is Exemption'.Value = "Yes",
            actor: If(varRecord.'Is Exemption'.Value = "Yes", varRecord.'L0 Approver', varRecord.'Level 1 Approver')},
        If(Radio1.Selected.Value = "Rejected",
            // ---------- REJECTED ----------
            If(isEx,
                Patch('PAS-Price Tracker', varRecord,
                    {Status: {Value:"Rejected"}, 'L0 Comments': Comments1.Text, 'L0 Status': {Value:"Rejected"}, 'Level 0 Date': Now()}),
                Patch('PAS-Price Tracker', varRecord,
                    {Status: {Value:"Rejected"}, 'Level 1 Comments': Comments1.Text,
                     'Level 1 Status': {Value:"Rejected"}, 'Level 1 Date': Now()}));
            Office365Outlook.SendEmailV2(
                varRecord.'Requestor Email',
                "Price Change Approval Rejected - " & varRecord.'Request ID',
                "Your request has been rejected by " & actor & ".<br>" &
                "Request ID: " & varRecord.'Request ID' & "<br><br>" &
                "Rejected by " & actor & "<br>" &
                "<b>Comments:</b> " & Comments1.Text
            ),
            // ---------- APPROVED ----------
            If(IsBlank(varRecord.'L2 Email'),
                // Contractual: final approval
                If(isEx,
                    Patch('PAS-Price Tracker', varRecord,
                        {Status: {Value:"Approved"}, 'L0 Comments': Comments1.Text, 'L0 Status': {Value:"Approved"}, 'Level 0 Date': Now()}),
                    Patch('PAS-Price Tracker', varRecord,
                        {Status: {Value:"Approved"}, 'Level 1 Comments': Comments1.Text,
                         'Level 1 Status': {Value:"Approved"}, 'Level 1 Date': Now()}));
                Office365Outlook.SendEmailV2(
                    varRecord.'Requestor Email' & If(isEx, ";" & varRecord.'L1 Email', ""),
                    "Request Approved - " & varRecord.'Request ID',
                    If(isEx,
                        "Your request has been fully approved by " & actor & ".<br><br>" &
                        "Request ID: " & varRecord.'Request ID' & "<br>" &
                        "Requestor: " & varRecord.Requestor & "<br>" &
                        "Plant: " & varRecord.Plant.Value & "<br>" &
                        "Commodity: " & varRecord.Commodity.Value & "<br>" &
                        "Sub-Commodity: " & varRecord.'Sub Commodity'.Value & "<br>" &
                        If(varRecord.'Sub Commodity'.Value = "Others", "ROB Type: " & varRecord.'ROB Category' & "<br>", "") &
                        "Contract Type: " & varRecord.'Contract Type'.Value & "<br>" &
                        "SAP Upload: " & Coalesce(varRecord.'SAP Upload Required'.Value, "No") & "<br>" &
                        If(!IsBlank(varRecord.Remarks), "Remarks: " & varRecord.Remarks & "<br>", "") &
                        "<br><b>Approver:</b> " & actor & "<br><b>Comments:</b> " & Comments1.Text,
                        "Request " & varRecord.'Request ID' & " is fully accepted.<br><br>" &
                        "Request ID: " & varRecord.'Request ID' & "<br>" &
                        "<b>Approver:</b> " & actor & "<br><b>Comments:</b> " & Comments1.Text
                    ),
                    {
                        Cc: varController &
                        With({L2: LookUp(
                            'PAS-Approver Master',
                            Plant.Value = varRecord.Plant.Value && 'Commodity.'.Value = varRecord.Commodity.Value)
                            .'L2 Approver'.Email}, If(!IsBlank(L2), ";" & L2, "")),
                        Attachments: varAttach,
                        IsHtml: true
                    }
                ),
                // Non-Contractual
                If(varRecord.'L1 Email' = varRecord.'L2 Email',
                    // L1 = L2: skip Level 2, go to Level 3
                    Patch('PAS-Price Tracker', varRecord,
                        {Status: {Value:"Pending L3"}, 'Current Approval Level': {Value:"3"},
                         'Level 1 Comments': Comments1.Text, 'Level 1 Status': {Value:"Approved"}, 'Level 1 Date': Now()});
                    Office365Outlook.SendEmailV2(
                        varRecord.'L3 Email',
                        "Action Needed: Price Change Approval " & varRecord.'Request ID' & " — Level 3 Approval",
                        "Hi " & varRecord.'Level 3 Approver' & ",<br><br>A request from " & varRecord.Requestor &
                        " has been approved at Level 1 (Level 2 skipped, same approver) and now needs your review.<br><br>" &
                        "Request ID: " & varRecord.'Request ID' & "<br>" &
                        "Requestor: " & varRecord.Requestor & "<br>" &
                        "Plant: " & varRecord.Plant.Value & "<br>" &
                        "Commodity: " & varRecord.Commodity.Value & "<br>" &
                        "Sub-Commodity: " & varRecord.'Sub Commodity'.Value & "<br>" &
                        If(varRecord.'Sub Commodity'.Value = "Others", "ROB Type: " & varRecord.'ROB Category' & "<br>", "") &
                        "Contract Type: " & varRecord.'Contract Type'.Value & "<br>" &
                        "SAP Upload: " & Coalesce(varRecord.'SAP Upload Required'.Value, "No") & "<br>" &
                        If(!IsBlank(varRecord.Remarks), "Remarks: " & varRecord.Remarks & "<br>", "") &
                        "<br><b>Approver:</b> " & actor & "<br><b>Comments:</b> " & Comments1.Text &
                        "<br><b>Level 1 Approver:</b> " & varRecord.'Level 1 Approver' &
                        "<br><b>Level 1 Comments:</b> " & Comments1.Text & "<br><br>" &
                        "Please review and approve/reject via the link below:<br>" &
                        "<a href=""<APP_LINK>"">Click here to review</a>",
                        {IsHtml: true,
                        Attachments: ForAll(
                            FileUpload.Attachments,
                            {
                                '@odata.type': "#Microsoft.OutlookServices.FileAttachment",
                                Name: ThisRecord.Name,
                                ContentBytes: ThisRecord.Value
                            })}),
                    // L1 ≠ L2: go to Level 2
                    Patch('PAS-Price Tracker', varRecord,
                        {Status: {Value:"Pending L2"}, 'Current Approval Level': {Value:"2"},
                         'Level 1 Comments': Comments1.Text, 'Level 1 Status': {Value:"Approved"}, 'Level 1 Date': Now()});
                    Office365Outlook.SendEmailV2(
                        varRecord.'L2 Email',
                        "Action Needed: Price Change Approval " & varRecord.'Request ID' & " — Level 2 Approval",
                        "Hi " & varRecord.'Level 2 Approver' & ",<br><br>A request from " & varRecord.Requestor &
                        " has been approved at Level 1 and now needs your review.<br><br>" &
                        "Request ID: " & varRecord.'Request ID' & "<br>" &
                        "Requestor: " & varRecord.Requestor & "<br>" &
                        "Plant: " & varRecord.Plant.Value & "<br>" &
                        "Commodity: " & varRecord.Commodity.Value & "<br>" &
                        "Sub-Commodity: " & varRecord.'Sub Commodity'.Value & "<br>" &
                        If(varRecord.'Sub Commodity'.Value = "Others", "ROB Type: " & varRecord.'ROB Category' & "<br>", "") &
                        "Contract Type: " & varRecord.'Contract Type'.Value & "<br>" &
                        "SAP Upload: " & Coalesce(varRecord.'SAP Upload Required'.Value, "No") & "<br>" &
                        If(!IsBlank(varRecord.Remarks), "Remarks: " & varRecord.Remarks & "<br>", "") &
                        "<br><b>Approver:</b> " & actor & "<br><b>Comments:</b> " & Comments1.Text &
                        "<br><b>Level 1 Approver:</b> " & varRecord.'Level 1 Approver' &
                        "<br><b>Level 1 Comments:</b> " & Comments1.Text & "<br><br>" &
                        "Please review and approve/reject via the link below:<br>" &
                        "<a href=""<APP_LINK>"">Click here to review</a>",
                        {IsHtml: true,
                        Attachments: ForAll(
                            FileUpload.Attachments,
                            {
                                '@odata.type': "#Microsoft.OutlookServices.FileAttachment",
                                Name: ThisRecord.Name,
                                ContentBytes: ThisRecord.Value
                            })}))));
        Notify("Response Submitted", NotificationType.Success);
        Reset(Comments1);
        Reset(Radio1);
        Refresh('PAS-Price Tracker');
        Navigate(scrHome))))
```

## How it works
1. **Collect the attachment:** The request's attached file is read
   from the gallery into `varAttach` for the final approval mail.

2. **Validate:** Comments and an Approved/Rejected choice are
   required, otherwise an error is shown.

3. **Load the request:** The request is looked up by its Request ID.
   The acting approver is L0 for exemption cases and L1 otherwise.

4. **Rejected:** The request status and that level's comments, status
   and date are saved. The requestor is emailed with the comments.

5. **Approved, Contractual:** There is no L2 for these requests, so
   this is the final approval. The status is set to Approved and the
   requestor is emailed (plus L1 for exemption cases), with the file
   attached and the controller and L2 approver in CC.

6. **Approved, Non-Contractual:** The request moves to the next level.
   If L1 and L2 are the same person, Level 2 is skipped and the request
   goes to L3. Otherwise it goes to L2. The status and current level
   are updated, and the next approver receives a summary with L1's
   comments, the file and a link to the approval screen.

7. **Finish:** A success message shows, the form is reset, the list is
   refreshed and the user returns to the home screen.
