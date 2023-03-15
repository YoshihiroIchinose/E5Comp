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

# Advance preparation
## 1. Prepare a SharePoint site to store external sharing operations
1. Create a dedicated SharePoint team site under the name "CustomeNotifcation"   
1. Create two blank lists from the site content of the site you created with the following names  
### AllowedDomains
Title-only list without any additional columns. Add domains as list items that you don't want to be notified of.
If the guest user's ID ends with these domains, the sharing activity for this guest will not be recorded.

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
  SharePointPnPPowerShellOnline  
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
$daysInterval=2

#Log retrieval range is 2 days as a test
$date=Get-Date
$Start=$date.addDays($daysInterval*-1).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
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

#Search results that are not on the list side are newly registered as list items.
#Add-PnPListItemのBatch処理だとUTCでタイムスタンプが書き込めなかったためCSOMを利用
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
#書き込みが多い場合には、一旦 100 アイテムで反映
  if($count % 100 -eq 0){
    $ctx.ExecuteQuery()}
}
$ctx.ExecuteQuery()
"File sharing activities were synched with the list."
Disconnect-PnPOnline
```

## 3. Power Automate によるメール通知の設定
1. SharingActivities のリストのメニューの統合から Power Automate を選択し、フローの作成を選択
1. "新しい SharePoint リスト アイテムが追加されたらカスタマイズされたメールを送信する"のフローを選択しフローを作成する
1. 編集から"Get My profile (V2)" および "Send Email" のステップを削除する
1. 新しいステップで "コントロール" の "条件" を追加する
1. 条件の値で、動的なコンテンツの "User" を指定し、"次の値を含む"、"#ext#" という条件を設定する   
   (共有操作を行ったのが外部ユーザーであれば、メール通知を本人に行わないようにするため)
1. "はいの場合" に "Outlook" の "メールの送信 (V2)" のアクションを追加する
1. 宛先に管理者のメールアドレスを設定する
1. 件名に "ゲスト ユーザーによる承認されていないドメインへの共有" と入力する
1. 本文におおよそ以下の内容を記載する([] は動的なコンテンツでリスト列を参照する)   
ゲスト ユーザーによる承認されていないドメインへの共有が行われました。   
内容を確認して下さい。   
ユーザー: [User]   
時間(UTC): [Time]   
共有されたアイテム: [SharedItem]   
共有先: [Guest]   

1. "いいえの場合" にも "Outlook" の "メールの送信 (V2)" のアクションを追加する
1. 宛先に動的なコンテンツでリスト列の "User" を指定する
1. 件名に"承認されていないドメインへの共有"と入力する
1. 本文におおよそ以下の内容を記載する([] は動的なコンテンツでリスト列を参照する)   
承認されていないドメインへの共有が行われました。意図しない共有の場合は、共有を解除ください。   
業務上必要な操作の場合には、Help Desk に連絡し、ドメインの許可を申請してください。  
ユーザー: [User]   
時間(UTC): [Time]   
共有されたアイテム: [SharedItem]   
共有先: [Guest]   

1. 後ろに新しいステップを追加し、"項目の更新" のアクションを追加する
1. サイトのアドレスで、"Customnotification" のサイトを指定し、リスト名で "SharingActivities" を指定する
1. ID を動的なコンテンツでリスト列の "ID" を指定する
1. タイトルを動的なコンテンツでリスト列の "タイトル" を指定する
1. "Notified" を "はい" に指定する
1. フローを保存する
送信されるメール通知は以下の通り。   
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Notification3.png"/>
その他、Power Automate を活用することで、ファイル共有操作をした上司を取得し、CC に上司を入れてメール通知することや、共有メールボックスを作成し、代理人として送信の権限を付与することで、共有メールボックスのアカウントを送信元としたメール通知も可能。
