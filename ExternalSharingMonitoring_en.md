# Email notification of unauthorized B2B external sharing operations
As a complement to DLP, I'd like to introduce a sample procedure that recognizes those operations from the audit log and notifies them by email
when internal users share files to external users in unauthorized domains through SharePoint Online, OneDriver for Business, Teams, etc.
The approximate mechanism of this operation is as follows.
1. Repare a SharePoint list to manage allowed domains
2. Repare a SharePoint list to record sharing operations
3. Set up an email notification to the user who performed the sharing operation triggered by an item creation in the SharPoint list via Power Automate 
4. Use Azure Automate to pull unauthorized share operations to external users of domains from the audit log at regular intervals and
write logs of new share operations to the SharePoint list while eliminating duplicates

Considerations
1. Azure Automate is not expensive but pay-as-you-go charges based on throughput
2. Power Automate email notifications are email notifications sent by the user who set them up, and Office 365 built-in licenses have a processing limit of 6,000 per day
3. If you want to provide timely notifications in units of several hours, the range of logs extracted 
by Azure Automate should also be limited to the range of the last few hours, and the frequency of schedule execution of Azure Automate should be several hours cycles.
It's ok that ranges of logs extracted by Auzre Automate overlap because this script elimantes duplicated activities by comparing recored activities in the SharePoint list
before it writes logs to the SharePoint list.
4. If you want to send another email notification for testing again, delete the item related to the sharing activities from the SharePoint list.
This ensures that when Azure Automate writes the log again in the SharePoint list, Power Automate sends an email notification for that item.

# Preparation
## 1. Prepare a SharePoint site to store external sharing operations
1. Create a dedicated SharePoint team site under the name "CustomeNotifcation"   
1. Create two blank lists from the site content of the site you created with the following names  
### AllowedDomains
Title-only list without any additional columns. Add domains as list items that you don't want to be notified of.
If the guest user's ID ends with these domains, the sharing activity for this guest will not be recorded.   
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Notification6.png"/>

### SharingActivities   
Add the following columns other than the title: To make it easier to handle in PowerShell, create columns with alphanumeric names as follows.      
| Column name | Title | User | Guest | Time | Operation | SharedItem | AdditionalData | Notified |
|-------|----|----|----|----|----|----|----|----|
| Column Type | Text | Text | Text | Date and time* | Text | Text | Multiple lines of text | Yes/No (Default No) |
| Information Stored | A GUID that identifies the log | User who shared | Guest with whom the item was shared | Time of sharing | Type of sharing | Shared item | Additional information | Flag whether notification processing has been performed |

*Pick a "Date and time" type and set Yes to "Include Time". Also, from the settings of the list of SharingActivities, set an index to "Time" in "Idexed columns" setting of the list.   
When logs are written to this list, it looks like the following.   
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Notification1.png"/>

## 2. Azure environment settings
1. Create an Azure Automation account with the name "LogNotification"     
1. In your Azure Automation account, go to "Modules" -> "Browse Gallery" and add the following three modules:   
(Runtime version is 5.1)   
  SharePointOnline.CSOM    
  PnP.PowerShell 
  ExchangeOnlineManagement   
1. In "Credentials" > "Add Credentials" for your Azure Automation account,
register the ID and password of an account with the name "Office 365" that has permission to extract audit logs and 
has permission to write to the specified SharePoint Online site.
1. In "Runbooks", "Create a runbook" for "PowerShell" with runtime version 5.1 named "ExtractSharingActivities"      
1. Copy and paste the following script into the runbook you created   
1. Edit the SharePoint site URL and list name in the script as appropriate, then save, and publish the Runbook
1. Start the runbook you created and check how it works   
1. Set up schedule execution such as daily as necessary    
When this script is executed in Azure Automation, the number of logs processed is output as follows.     
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Notification2.png"/>

#### Aure Automation Sample Script
```
#Variables which should be modified based on the environment
$Credential = Get-AutomationPSCredential -Name "Office 365"
$SiteUrl="https://xxxx.sharepoint.com/sites/CustomNotification/"
$AllowedDomainList="AllowedDomains"
$SharingActivitiesList="SharingActivities"
$HoursInterval=48

#Log retrieval range is 48 hours as a test
$date=Get-Date
$Start=$date.addHours($HoursInterval*-1).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$End=$date.ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")

#Get a domain allowlist
Connect-PnPOnline -Url $SiteUrl -credentials $Credential
$AllowedDmains=@()
foreach($item in Get-PnPListItem -list $AllowedDomainList -PageSize 1000){
    $AllowedDmains+=$item.FieldValues["Title"].ToLower()
}

#A function for adding a member to the object
Function AddMember{
    Param($a,$b,$c)
    add-member -InputObject $a -NotePropertyName $b  -NotePropertyValue $c
}

#A function that makes the Azure AD B2B guest ID notation easier to read
Function ExtractGuest{
    Param($a)
    $a=$a.replace("#EXT#","#ext#")
    If($a.Contains("#ext#")){
    return $a.Substring(0,$a.IndexOf("#ext#")).replace("_","@")
    }
    return $a
}

#A function to determine if a domain is on the allowlist
Function IsAllowed{
    Param($a)
    $a=$a.ToLower()
    foreach($i in $AllowedDmains){
        if($a.EndsWith($i)){return $true}
    }
    return $false
}

#1. Get logs of operations that add guest users to SharePoint groups and grant permissions
Connect-ExchangeOnline -credential $Credential
$RecordType="SharePointSharingOperation"
$Operation="AddedToGroup"
$output=Search-UnifiedAuditLog -RecordType $RecordType -StartDate $Start -EndDate $End -Operations $Operation -ResultSize 5000 -SessionCommand ReturnNextPreviewPage|?{$_.UserIds -ne "app@sharepoint"}
#If the result is not null, consider the number of result is one at first, and if multiple results are returned, get the count
$count=0
if($output -ne $null){$count=1}
if($output.count -ne $null){$count=$output.count}
"Site sharing activities logs since $Start"+": "+$count

$csv=@()
foreach($i in $output){
$AuditData=$i.AuditData|ConvertFrom-Json
#Excludes non-guest additions and guest additions from allowed domains
If($AuditData.TargetUserOrGroupType -ne "Guest" ){continue}
$guest=ExtractGuest $AuditData.TargetUserOrGroupName
If(isAllowed($guest)){continue}

$line = New-Object -TypeName PSObject
AddMember $line "User" $i.UserIds
AddMember $line "Guest" $guest
AddMember $line "Time" $i.CreationDate.ToString("yyyy-MM-ddTHH:mm:ssZ")
AddMember $line "Operation" "Site Shared"
AddMember $line "SharedItem" $AuditData.SiteUrl
AddMember $line "LogId" $AuditData.CorrelationId
AddMember $line "AdditionalData" $AuditData.EventData
$csv+=$line
}
"Unallowed site sharing activities since $Start"+": "+$csv.count

#Merge logs with the same CorrelationID
$GroupedCsv=@()
foreach($i in ($csv|Group-Object LogId)){
    $line = $i.Group[0]
    $AdditionalData=@()
    foreach($d in $i.Group){
        $AdditionalData+=$d.AdditionalData
    }
    $line.AdditionalData=$AdditionalData -join "`r`n"
$GroupedCsv+=$line
}
"Unallowed site sharing activities merged since $Start"+": "+$GroupedCsv.count

#2. Get logs of operations that share files directly to guest users
$RecordType="SharePointSharingOperation"
$Operation="AddedToSecureLink"
$output=Search-UnifiedAuditLog -RecordType $RecordType -StartDate $Start -EndDate $End -Operations $Operation -ResultSize 5000 -SessionCommand ReturnNextPreviewPage
#If the result is not null, consider the number of result is one at first, and if multiple results are returned, get the count
$count=0
if($output -ne $null){$count=1}
if($output.count -ne $null){$count=$output.count}
"File sharing activities logs since $Start"+": "+$count

$count=0
foreach($i in $output){
$AuditData=$i.AuditData|ConvertFrom-Json
#Excludes non-guest additions and guest additions from allowed domains
If($AuditData.TargetUserOrGroupType -ne "Guest" ){continue}
$guest=ExtractGuest $AuditData.TargetUserOrGroupName
If(isAllowed($guest)){continue}

$line = New-Object -TypeName PSObject
AddMember $line "User" $i.UserIds
AddMember $line "Guest" $guest
AddMember $line "Time" $i.CreationDate.ToString("yyyy-MM-ddTHH:mm:ssZ")
AddMember $line "Operation" "File Shared"
AddMember $line "SharedItem" $AuditData.ObjectId
AddMember $line "LogId" $AuditData.CorrelationId
AddMember $line "AdditionalData" $AuditData.EventData
$GroupedCsv+=$line
$count++
}
"Unallowed file sharing activities since $Start"+": "+$count

#Merge logs of 1 and 2 with the same CorrelationID while prioritizing 2, "File Shared"
$GroupedCsv2=@()
foreach($i in ($GroupedCsv|Group-Object LogId)){
    $line = $i.Group[0]
　　$AdditionalData=@()
    foreach($j in $i.Group){
		$AdditionalData+=$j.AdditionalData
	    if($j.Operation -eq "File Shared"){$line=$j}
    }
    $line.AdditionalData=$AdditionalData -join "`r`n"
    $GroupedCsv2+=$line
}
"Unallowed total sharing activities since $Start"+": "+$GroupedCsv2.count

#3.Get logs for adding a guest user to an existing group
$RecordType="AzureActiveDirectory"
$Operation="Add member to group."
$output=Search-UnifiedAuditLog -RecordType $RecordType -StartDate $Start -EndDate $End -Operations $Operation
#If the result is not null, consider the number of result is one at first, and if multiple results are returned, get the count
$count=0
if($output -ne $null){$count=1}
if($output.count -ne $null){$count=$output.count}
"Adding member activities since $Start"+": "+$count
Disconnect-ExchangeOnline -Confirm:$false

$count=0
foreach($i in $output){
    $AuditData=$i.AuditData|ConvertFrom-Json
    #Excludes non-guest additions
    If(!$AuditData.ObjectId.Contains("#EXT#")){continue}
    $guest=ExtractGuest $AuditData.ObjectId
    If(isAllowed($guest)){continue}

    $line = New-Object -TypeName PSObject
    AddMember $line "User" $i.UserIds
    AddMember $line "Guest" $guest
    AddMember $line "Time" $i.CreationDate.ToString("yyyy-MM-ddTHH:mm:ssZ")
    AddMember $line "Operation" "Guest Added to Group"
    AddMember $line "SharedItem" $AuditData.ModifiedProperties[1].NewValue
    AddMember $line "LogId" $AuditData.ID
    AddMember $line "AdditionalData" $AuditData.ModifiedProperties[0].NewValue
    $count++
    $GroupedCsv2+=$line
}
"Unallowed adding member activities since $Start"+": "+$count

#Remove duplicates of current list items and do not upload them
$CAML="<Query><Where><Geq><FieldRef Name='Time'/><Value Type='DateTime' IncludeTimeValue='TRUE'>$Start</Value></Geq></Where></Query>"
$GroupedCsv2 = {$GroupedCsv2}.Invoke()
foreach($item in (Get-PnPListItem -list $SharingActivitiesList -PageSize 1000 -Query $CAML)){
    $target=-1
    for($i=0;$i -lt $GroupedCsv2.count;$i++){
        if($GroupedCsv2[$i].LogId -eq $item.FieldValues["Title"]){
            $target=$i
            continue
        }
    }
    if($target -ge 0){
        $GroupedCsv2.RemoveAt($target)
    }
}
"Newly identified total sharing activities since $Start"+": "+$GroupedCsv2.count

#Write logs that are not on the SharePoint List
#In the Batch process of Add-PnPListItem, the timestamp could not be written in UTC, so use CSOM.
$ctx=get-pnpcontext
$list = $ctx.get_web().get_lists().getByTitle($SharingActivitiesList)
$count=0
foreach($item in $GroupedCsv2){
$lic = New-Object Microsoft.SharePoint.Client.ListItemCreationInformation
$i = $list.AddItem($lic)
$i.set_item("Title", $item.LogId)
$i.set_item("User", $item.User)
$i.set_item("Guest", $item.Guest)
$i.set_item("Time", $item.Time)
$i.set_item("Operation", $item.Operation)
$i.set_item("SharedItem", $item.SharedItem)
$i.set_item("AdditionalData", $item.AdditionalData)
$i.update()
$count++
#If there are many writes, reflect them per 100 items
  if($count % 100 -eq 0){
    $ctx.ExecuteQuery()}
}
$ctx.ExecuteQuery()
"File sharing activities were synched with the list."
Disconnect-PnPOnline
```

## 3. Set up email notifications with Power Automate
1. Select Power Automate from the "Integrate" in the menu of SharingActivities list, and then select "Create a flow"
1. Create a flow by selecting the "Send a customized email when a new SharePoint list item is added" flow
1. Edit a flow and Remove the "Get My profile (V2)" and "Send an email" steps
1. Add a "Condition" in a "Control"category as a new step
1. In the condition value, specify "User" for Dynamic content, and set the condition "contains" and "#ext#" as a value 
   (If the sharing operation was performed by an external user, to prevent the user from receiving email notifications)
1. Add the "Send an email (V2)" action in "Office 365 Outlook" category in "If yes" section
1. In the "To", type the administrator's fixed email address
1. In the "Subject", type "Guest user sharing to unauthorized domain"
1. Approximately include the following in the "Body" ([] refers to the list column in dynamic content)   
A guest user shared to an unauthorized domain.   
Please check the contents. 
User: [User]   
Time(UTC): [Time]   
SharedItem: [SharedItem]   
Share with: [Guest]   

1. Add "Get manager (V2)" action in "If No" section
1. In the "User (UPN)", add [User] from the list column as dynamic content
1. Add the "Send an email (V2)" action in "Office 365 Outlook" category in "If No" section as well
1. In the "To", add [User] in the list column as dynamic content 
1. In the "Subject", type "Sharing to unauthorized domains"
1. Approximately include the following in the "Body" ([] refers to the list column in dynamic content)   
Sharing to an unauthorized domain occurred. If it's an unintended sharing, unshare it.   
If it is necessary for business purposes, please contact Help Desk to request permission for the domain.   
User: [User]   
Time(UTC): [Time]   
SharedItem: [SharedItem]   
Share with: [Guest]   

1. Add a new step at the last and add an "Update Item" action
1. Specify the site for "Customnotification" in the site address and "SharingActivities" in the list name
1. Specify the [ID] of a list column as dynamic content
1. Specify the [title] of a list column as dynamic content
1. Specify "Notified" as "Yes"
1. Save the flow   

## The overview of this flow should be like below.    
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Notification5.png"/>

## The email notifications sent by this flow are as follows. 
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Notification4.png"/>
In addition, by utilizing Power Automate, it is possible to send e-mail notifications from the account of the shared mailbox by creating a shared mailbox and granting the permission to send as a proxy.
