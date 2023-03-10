# Azure Automation を利用して Office 365 の監査ログから B2B 外部共有操作のログを SharePoint List に書き出し、Power Automate でメール通知する

# 事前準備
## 1. 外部共有操作を書き出す SharePoint サイトの準備
1. 専用の SharePoint チーム サイトを CustomeNotifcation の名称で作成する   
1. 作成したサイトのサイト コンテンツから以下の名称で 2 つの空白のリストを作成する   
### AllowedDomains
特に列の追加なくタイトルのみのリスト。通知対象外とするドメインを追加しておく。ゲスト ユーザーの　ID と後方一致で除外とする。
### SharingActivities   
タイトル以外の以下の列を追加する。PowerShell で扱いやすくするために、以下の通り英数字で列は作成のこと。   
| 列の名称 | タイトル | User | Guest | Time | Operation | SharedItem | AdditionalData | Notified |
|-------|----|----|----|----|----|----|----|----|
| 列の種類 | 既定の 1 行テキスト | 1 行テキスト | 1 行テキスト | 日付と時刻* | 1 行テキスト | 1 行テキスト | 複数行テキスト | はい/いいえ (既定値 いいえ) |
| 格納する情報 | ログを識別する GUID | 操作を行ったユーザー | 招待されたゲスト | 招待した時間 | 操作内容の種別 | 共有されたアイテム | その他付随情報を格納 | 通知処理を行ったかどうかのフラグ |

*日付と時刻の種類を選び、時刻を含めるをはいとする。   
c. SharingActivities のリストの設定から、列のインデックス付きの列で、Time にインデックスを設定しておく。

## 2. Azure 環境の設定
1. Azure Automation アカウントを作成   
1. Azure Automation アカウントにて、"モジュール" -> "ギャラリーを参照"から、以下の 3 つのモジュールを追加する。   
(ランタイム バージョンは 5.1)   
  SharePointOnline.CSOM    
  SharePointPnPPowerShellOnline  
  ExchangeOnlineManagement   
1. Azure Automation アカウントの"資格情報"->"資格情報の追加"で、監査ログの抽出権限があり、   
  指定の SharePoint Online サイトに投稿権限があるアカウントの ID とパスワードを "Office 365" という名称で登録しておく。
1. Azure Automation アカウントの"Runbook"->"Runbook の作成"で PowerShell、ランタイム バージョンの 5.1 の Runbook を作成する   
1. 作成した Runbook に以下のスクリプトをコピー & ペーストする   
1. 適宜スクリプト内の SharePoint Site の URL および、リスト名を変更・保存し、公開する   
1. 作成した Runbook を"開始"し、動作を確認する   
1. 必要に応じて Daily 等のスケジュール実行を設定する   

#### Aure Automation サンプル スクリプト
```
#変数
$Credential = Get-AutomationPSCredential -Name "Office 365"
$SiteUrl="https://xxxx.sharepoint.com/sites/CustomNotification/"
$AllowedDomainList="AllowedDomains"
$SharingActivitiesList="SharingActivities"
$daysInterval=2

#ログの取得範囲は 2 日間の範囲
$date=Get-Date
$Start=$date.addDays($daysInterval*-1).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$End=$date.ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")

#ドメイン許可リストの取得
Connect-PnPOnline -Url $SiteUrl -credentials $Credential
$AllowedDmains=@()
foreach($item in Get-PnPListItem -list $AllowedDomainList -PageSize 1000){
    $AllowedDmains+=$item.FieldValues["Title"].ToLower()
}

#オブジェクトに値を設定する Function
Function AddMember{
    Param($a,$b,$c)
    add-member -InputObject $a -NotePropertyName $b  -NotePropertyValue $c
}

#Azure AD B2B のゲスト ID の表記を見やすくする Function
Function ExtractGuest{
    Param($a)
    $a=$a.replace("#EXT#","#ext#")
    If($a.Contains("#ext#")){
    return $a.Substring(0,$a.IndexOf("#ext#")).replace("_","@")
    }
    return $a
}

#ドメイン許可リストに含まれているか判定する Funciton
Function IsAllowed{
    Param($a)
    $a=$a.ToLower()
    foreach($i in $AllowedDmains){
        if($a.EndsWith($i)){return $true}
    }
    return $false
}

#1. ゲスト ユーザーを SharePoint グループに追加し権限を付与する操作のログの取得
Connect-ExchangeOnline -credential $Credential
$RecordType="SharePointSharingOperation"
$Operation="AddedToGroup"
$output=Search-UnifiedAuditLog -RecordType $RecordType -StartDate $Start -EndDate $End -Operations $Operation -ResultSize 5000 -SessionCommand ReturnNextPreviewPage|?{$_.UserIds -ne "app@sharepoint"}
#結果がNullでなければ、まずは結果は1個とし、結果が複数個返されている場合には、そのカウントを取得
$count=0
if($output -ne $null){$count=1}
if($output.count -ne $null){$count=$output.count}
"Site sharing activities logs since $Start"+": "+$count

$csv=@()
foreach($i in $output){
$AuditData=$i.AuditData|ConvertFrom-Json
#ゲスト以外の追加や、許可されたドメインのゲスト追加は除く
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

#同じ CorrelationID のログをマージ
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

#2.ファイルを直接ゲスト ユーザーに共有する操作のログの取得
$RecordType="SharePointSharingOperation"
$Operation="AddedToSecureLink"
$output=Search-UnifiedAuditLog -RecordType $RecordType -StartDate $Start -EndDate $End -Operations $Operation -ResultSize 5000 -SessionCommand ReturnNextPreviewPage
#結果がNullでなければ、まずは結果は1個とし、結果が複数個返されている場合には、そのカウントを取得
$count=0
if($output -ne $null){$count=1}
if($output.count -ne $null){$count=$output.count}
"File sharing activities logs since $Start"+": "+$count

$count=0
foreach($i in $output){
$AuditData=$i.AuditData|ConvertFrom-Json
#ゲスト以外への共有や、許可されたドメインのゲストへの共有は除く
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

#1と2で共通する CorrelationID のログをFile Shared 優先でマージ
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

#3.既存グループへのゲスト ユーザーの追加操作のログの取得
$RecordType="AzureActiveDirectory"
$Operation="Add member to group."
$output=Search-UnifiedAuditLog -RecordType $RecordType -StartDate $Start -EndDate $End -Operations $Operation
#結果がNullでなければ、まずは結果は1個とし、結果が複数個返されている場合には、そのカウントを取得
$count=0
if($output -ne $null){$count=1}
if($output.count -ne $null){$count=$output.count}
"Adding member activities since $Start"+": "+$count
Disconnect-ExchangeOnline -Confirm:$false

$count=0
foreach($i in $output){
    $AuditData=$i.AuditData|ConvertFrom-Json
    #ゲスト以外の追加は除く
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

#現在のリスト アイテムと重複するものは削除してアップロードしない
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

#リスト側にない検索結果はリストアイテムとして新規に登録する
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
1. 編集から"Get My profile (V2)"のステップを削除する
2. Send Email のステップで、宛先に動的なコンテンツのもっと見るからリスト列である User を選択する
3. 件名に"承認されていないドメインへの共有"を入力する
4. 本文におおよそ以下の内容を記載する([]動的なコンテンツでリスト列を参照する)   
[User]により、[Time]に、[SharedItem]が、
未承認のドメインのユーザーである[Guest]に共有されました。
意図しない共有の場合は、共有を解除ください。
業務上必要な操作の場合には、Help Desk に連絡し、ドメインの許可を申請してください。   
5. フローを保存する
