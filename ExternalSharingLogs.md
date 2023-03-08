# B2B コラボレーションを中心とした外部共有のログの確認
各種 Office 365 の SharePoint Online や OneDrive for Business の DLP 機能で、外部に機密情報等を共有した際、ブロックや警告を行うことは可能です。
ただ、DLP では、どういったファイルが DLP のポリシーに合致したかまでは把握することができるものの、メールを除いて、
どのドメインの外部ユーザーに共有したかまでは具体的にトラッキングしないため、外部共有を許可する前提の場合、ログから外部共有先を把握したいというニーズもあるかと思います。
そこで、このスクリプトでは、以下の 3 つの操作を対象に共有先のゲスト ユーザーを含め外部共有に関するログを CSV 形式で抽出します。
1. SharePoint Online / OneDriver for Business のサイトでゲスト ユーザーを SharePoint グループに追加し権限を付与する操作
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Log1.png">
2. SharePoint Online / OneDriver for Business でファイルを直接ゲスト ユーザーに共有する操作
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Log2.png">
3. 既存グループへのゲスト ユーザーの追加
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/Log3.png">

ただし、上記ログは、Azure AD B2B Federation を前提とした操作のみを対象としているため、
B2B Direct を利用する Teams Connect や、匿名リンクなどは対象外となります。

## スクリプト
```
#接続に利用するID / パスワード
$Id="xxx@xxx.onmicrosoft.com"
$Password = "xxxxx"

#変数
$Startdate="2023/01/01"
$Enddate="2023/03/08"
$OutputFolder=[System.Environment]::GetFolderPath("Desktop")+"\"

#Credentialの生成
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Import-Module ExchangeOnlineManagement
Connect-ExchangeOnline -credential $Credential

#1. ゲスト ユーザーを SharePoint グループに追加し権限を付与する操作のログ
$RecordType="SharePointSharingOperation"
$Operation="AddedToGroup"
$output=Search-UnifiedAuditLog -RecordType $RecordType -StartDate $Startdate -EndDate $Enddate -Operations $Operation -ResultSize 5000 -SessionCommand ReturnNextPreviewPage|?{$_.UserIds -ne "app@sharepoint"}

$csv=@()
foreach($row in $output){
$JsonRaw=$row.AuditData|ConvertFrom-Json
#ゲスト以外の追加は除く
If($JsonRaw.TargetUserOrGroupType -ne "Guest" ){continue}
$data=$row |select UserIds,Operations,CreationDate
add-member -InputObject $data -NotePropertyName CorrelationId  -NotePropertyValue $JsonRaw.CorrelationId
add-member -InputObject $data -NotePropertyName SiteUrl  -NotePropertyValue $JsonRaw.SiteUrl
add-member -InputObject $data -NotePropertyName EventData  -NotePropertyValue $JsonRaw.EventData
$GuestId=$JsonRaw.TargetUserOrGroupName
If($GuestId.Contains("#ext#"))
	{$GuestId=$GuestId.Substring(0,$GuestId.IndexOf("#ext#")).replace("_","@")}
add-member -InputObject $data -NotePropertyName TargetUserOrGroupName  -NotePropertyValue $GuestId
$csv+=$data
}

#CSV ファイルとして出力
$filePath=$OutputFolder+"SPO_AddedToGroup"+(Get-Date -Format "yyyyMMdd-HHmm")+".csv"
$csv|Export-csv -Path $filePath  -Encoding UTF8 -NoTypeInformation

#2.ファイルを直接ゲスト ユーザーに共有する操作のログ
$RecordType="SharePointSharingOperation"
$Operation="AddedToSecureLink"
$output=Search-UnifiedAuditLog -RecordType $RecordType -StartDate $Startdate -EndDate $Enddate -Operations $Operation -ResultSize 5000 -SessionCommand ReturnNextPreviewPage

$csv=@()
foreach($row in $output){
$JsonRaw=$row.AuditData|ConvertFrom-Json
#ゲスト以外への共有は除く
If($JsonRaw.TargetUserOrGroupType -ne "Guest" ){continue}
$data=$row |select UserIds,Operations,CreationDate
add-member -InputObject $data -NotePropertyName CorrelationId  -NotePropertyValue $JsonRaw.CorrelationId
add-member -InputObject $data -NotePropertyName FileUrl  -NotePropertyValue $JsonRaw.ObjectId
add-member -InputObject $data -NotePropertyName EventData  -NotePropertyValue $JsonRaw.EventData
$GuestId=$JsonRaw.TargetUserOrGroupName
If($GuestId.Contains("#ext#"))
	{$GuestId=$GuestId.Substring(0,$GuestId.IndexOf("#ext#")).replace("_","@")}
add-member -InputObject $data -NotePropertyName TargetUserOrGroupName  -NotePropertyValue $GuestId
$csv+=$data
}

#CSV ファイルとして出力
$filePath=$OutputFolder+"SPO_AddedToSecureLink"+(Get-Date -Format "yyyyMMdd-HHmm")+".csv"
$csv|Export-csv -Path $filePath  -Encoding UTF8 -NoTypeInformation

#3.既存グループへのゲスト ユーザーの追加操作のログ
$RecordType="AzureActiveDirectory"
$Operation="Add member to group."
$output=Search-UnifiedAuditLog -RecordType $RecordType -StartDate $Startdate -EndDate $Enddate -Operations $Operation

$csv=@()
foreach($row in $output){
$JsonRaw=$row.AuditData|ConvertFrom-Json
#ゲスト以外の追加は除く
If(!$JsonRaw.ObjectId.Contains("#EXT#")){continue}
$data=$row |select UserIds,Operations,CreationDate
add-member -InputObject $data -NotePropertyName LogID  -NotePropertyValue $JsonRaw.ID
add-member -InputObject $data -NotePropertyName GroupName  -NotePropertyValue $JsonRaw.ModifiedProperties[1].NewValue
add-member -InputObject $data -NotePropertyName GroupID  -NotePropertyValue $JsonRaw.ModifiedProperties[0].NewValue
$AddedGuest=$JsonRaw.ObjectId.Substring(0,$JsonRaw.ObjectId.IndexOf("#EXT#")).replace("_","@")
add-member -InputObject $data -NotePropertyName AddedGuest  -NotePropertyValue $AddedGuest
$csv+=$data
}
$csv|Export-csv -Path ($OutputFolder+"AAD_AddedToGroup"+(Get-Date -Format "yyyyMMdd-HHmm")+".csv") -Encoding UTF8 -NoTypeInformation

Disconnect-ExchangeOnline -Confirm:$false
```
