# 秘密度ラベル操作をメール通知する
秘密度ラベルの変更に伴う情報漏えいを監視するため、M365 Apps、SharePoint (Office for the web)、AIP アドインのそれぞれの監査ログから、
秘密ラベルのダウングレード・秘密度ラベルの削除・サブ ラベルの変更による暗号化解除の操作を抜き出し、これら操作のログを、
SharePoint リストに書き込みと共に、Azure Automate を利用して、カスタムのメール通知を行うソリューションの実装サンプルになります。

おおよその動作の仕組みとしては以下の通りです。
1. ラベル操作を記録する SharePoint リストを用意しておく
2. 上記 SharePoint リストに Power Automate で新しいアイテムが追加された際、ラベル操作を行ったユーザーにメール通知を行う処理を設定しておく
3. Azure Automate を用いて、一定期間ごとに監査ログから特定のラベル操作を抜き出し、重複を排除しながらログを 1 のリストに書き込む

考慮事項
1. Azure Automate では高額ではないが処理量に応じた従量課金が発生
2. Power Automate のメール通知では、設定を行ったユーザーが送信者となったメール通知となり、Office 365 組み込みのライセンスでは、1 日 6,000 件といった処理量の上限あり
3. 重複を排除したラベル操作の追記をリストに行うため、Auzre Automate で抜き出すログの範囲は重複してもよい。もし数時間単位でタイムリーに通知を行いたい場合、Azure Automate で抜き出すログの範囲も直近数時間といった範囲に狭め、スケジュール実行の頻度を数時間サイクルとすること。
4. テスト用に再度メール通知を行いたい場合には、SharePoint のリストから、該当のラベル操作のログを消すこと。これにより Azure Automate で再度ログが登録しなおされた際、メール通知が Power Automate により行われる。

# 事前準備
## 1. ラベル操作を書き出す SharePoint サイトの準備
1. 専用の SharePoint チーム サイトを CustomeNotifcation の名称で作成する   
1. 作成したサイトのサイト コンテンツから以下の名称で空白のリストを作成する   
### LabelActivities
タイトル以外の以下の列を追加する。PowerShell で扱いやすくするために、以下の通り英数字で列は作成する。   
| 列の名称 | タイトル | User | Time | Item | Operation | CurrentLabel | PreviousLabel | EncryptionStatusChange | Justification | Workload | Notified |   
|-------|----|----|----|----|----|----|----|----|----|----|----|   
| 列の種類 | 既存の 1 行テキスト | 1 行テキスト | 日付と時刻* | 1 行テキスト | 1 行テキスト | 1 行テキスト | 1 行テキスト | 1 行テキスト | 1 行テキスト | 1 行テキスト | はい/いいえ (既定値 いいえ) |
| 格納する情報 | ログを識別する GUID | 操作を行ったユーザー | 操作した時間 | 対象のファイル |ラベル操作内容 | 現在の秘密度ラベル | 以前の秘密度ラベル | 暗号化状態の変化 | ラベル変更の理由 | ラベル変更を行ったアプリ |通知処理を行ったかどうかのフラグ |

*日付と時刻の種類を選び、時刻を含めるをはいとする。また LabelActivities のリストの設定から、列のインデックス付きの列で、Time にインデックスを設定しておく。   
こちらのリストにログが書き込まれると以下のような見た目となる。   
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/LabelNotif01.png"/>

## 2. Azure 環境の設定
1. Azure Automation アカウントを作成   
1. Azure Automation アカウントにて、"モジュール" -> "ギャラリーを参照"から、以下の 3 つのモジュールを追加する。   
(ランタイム バージョンは 5.1)   
  SharePointOnline.CSOM    
  PnP.PowerShell  
  ExchangeOnlineManagement   
1. Azure Automation アカウントの"資格情報"->"資格情報の追加"で、監査ログの抽出権限があり、   
  指定の SharePoint Online サイトに投稿権限があるアカウントの ID とパスワードを "Office 365" という名称で登録しておく。
1. Azure Automation アカウントの"Runbook"->"Runbook の作成"で PowerShell、ランタイム バージョン 5.1 の Runbook を作成する   
1. 作成した Runbook に以下のスクリプトをコピー & ペーストする   
1. 適宜スクリプト内の SharePoint Site の URL および、リスト名を変更・保存し、公開する   
1. 作成した Runbook を"開始"し、動作を確認する   
1. 必要に応じて Daily 等のスケジュール実行を設定する   
Azure Automation で本スクリプトを実行すると、以下のように処理されたログの件数が出力される。   
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/LabelNotif02.png"/>

#### Aure Automation サンプル スクリプト

```
$Credential = Get-AutomationPSCredential -Name "Office 365"
$SiteUrl="https://xxxx.sharepoint.com/sites/CustomNotification"
$LabelActivitiesList="LabelActivities"
$HoursInterval=48
#ログの取得範囲はテスト環境として過去 48 時間の範囲
$date=Get-Date
$Start=$date.addHours($HoursInterval*-1).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$End=$date.ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")

#オブジェクトに値を設定する Function
Function AddMember{
    Param($a,$b,$c)
    add-member -InputObject $a -NotePropertyName $b  -NotePropertyValue $c
}

#秘密度ラベルを GUID から表示名に変換する Function
Function GetLabelName{
    Param($a)
    if($a -eq $null){return $null}
    foreach($l in $labels){if($l.Guid -eq $a){return $l.Name}}
    return "Not Found"
}

#GUID で指定されたラベルが、暗号化をするラベルか判定する Function
Function IsEncryptionLabel{
    Param($a)
    if($a -eq $null){return $false}
    foreach($l in $labels){
      if($l.Guid -ne $a){continue}
      foreach($act in $l.LabelActions){
        $j=convertfrom-json $act
        If($j.Type -eq "encrypt"){return $true}
      }
    }
    return $false
}

#監査ログを 5,000 件 x 最大 10 回で 50,000 件取得し、$global:output に格納する Function
Function ExtractAuditLog{
  Param($type,$op)
  if($type -eq $null){return}
  $itemcount=0
	for($i = 0; $i -lt 10; $i++){
        $result=Search-UnifiedAuditLog -RecordType $type -StartDate $global:Start -EndDate $global:End -SessionId $type -Operations $op	-SessionCommand ReturnLargeSet -ResultSize 5000
        "Query for $type, Round("+($i+1)+"): "+$result.Count.ToString() + " items"
        $global:output+=$result
        $itemcount+=$result.Count
        if($result.count -ne 5000){break}
    }
    "$type Total: "+$itemcount.ToString() + " items"
}

#3 M365Apps, SharePoin(WAC), AIP の種類によらず $global:output からラベル操作の内容を抽出し、$global:csv に書き出す
Function FormatLabelActitivyLog {
  foreach($i in $global:output){
    $Operation=""
    $EncryptionStatusChange=""
    $AuditData=$i.AuditData|ConvertFrom-Json
    $Operation=switch($AuditData.SensitivityLabelEventData.LabelEventType){
      1 {"LabelUpgraded"}
      2 {"LavelDowngraded"}
      3 {"LavelRemoved"}
      4 {"LabelChangedSameOrder"}
    }
    #ラベルのアップグレードは除外
    if($Operation -eq "LabelUpgraded"){continue}
    
    #M365 Apps / AIP のいずれかで現在暗号化されているか
    $isEncrypted=$AuditData.IrmContentId -ne $null -or $AuditData.ProtectionEventData.IsProtected
    #Office for the web の場合はラベルの設定から暗号化を判断
    $isEncrypted=$isEncrypted -or ($AuditData.UserAgent -eq "MSWAC" -and (IsEncryptionLabel($AuditData.SensitivityLabelEventData.SensitivityLabelId)))
	  #以前暗号化されていたか判定。AIP の場合、暗号化フラグを参照
    $wasEncrypted=$AuditData.Workload -eq "AIP" -and $AuditData.ProtectionEventData.IsProtectedBefore
    #AIP 以外の場合、以前のラベルの秘密度ラベル設定から暗号化有無を判断
    $wasEncrypted=$wasEncrypted -or (IsEncryptionLabel($AuditData.SensitivityLabelEventData.OldSensitivityLabelId))
    #暗号化の変更状況を判断
    if($isEncrypted){
      If($wasEncrypted){$EncryptionStatusChange="NoChangeWithEncryption"}
      Else {$EncryptionStatusChange="NewlyEncrypted"}
    }
    Else{
      If($wasEncrypted){$EncryptionStatusChange="EncryptionRemoved"}
      Else {$EncryptionStatusChange="NoChangeWithoutEncryption"}
    }
    
    #同一の機密度のサブ ラベル変更は、暗号化が削除されたケースのみを対象にする
    if($Operation -eq "LabelChangedSameOrder" -and $Operation -ne "EncryptionRemoved"){continue}
	
	  $line = New-Object -TypeName PSObject
    AddMember $line "LogId" $Auditdata.Id
    AddMember $line "User" $i.UserIds
    AddMember $line "Time" $i.CreationDate.ToString("yyyy-MM-ddTHH:mm:ssZ")
    AddMember $line "Item" $AuditData.ObjectId
    AddMember $line "Operation" $Operation
    AddMember $line "CurrentLabel" (GetLabelName($AuditData.SensitivityLabelEventData.SensitivityLabelId))
    AddMember $line "PreviousLabel" (GetLabelName($AuditData.SensitivityLabelEventData.OldSensitivityLabelId))
    AddMember $line "EncryptionStatusChange" $EncryptionStatusChange
    AddMember $line "Justification" ($AuditData.SensitivityLabelEventData.JustificationText + $AuditData.SensitivityLabelJustificationText)
    AddMember $line "Workload" $AuditData.Workload
    $global:csv+=$line
    }
}

#ラベル情報の取得
Connect-IPPSsession -credential $credential
$labels=get-label

#Exchange Online に接続
Connect-ExchangeOnline -credential $credential
$csv=@()

#M365 Apps Native
$output=@()
ExtractAuditLog "SensitivityLabelAction" "SensitivityLabelRemoved,SensitivityLabelUpdated" 
FormatLabelActitivyLog

#Office for the web (SharePoint)
$output=@()
ExtractAuditLog "SharePointFileOperation" "FileSensitivityLabelChanged,FileSensitivityLabelRemoved"
FormatLabelActitivyLog

#AIP Client
$output=@()
ExtractAuditLog "AipSensitivityLabelAction" "SensitivityLabelRemoved,SensitivityLabelUpdated"
FormatLabelActitivyLog

Disconnect-ExchangeOnline -Confirm:$false

#現在のリスト アイテムと重複するものは削除してアップロードしない
Connect-PnPOnline -Url $SiteUrl -credentials $Credential
$CAML="<Query><Where><Geq><FieldRef Name='Time'/><Value Type='DateTime' IncludeTimeValue='TRUE'>$Start</Value></Geq></Where></Query>"
$csv2 = {$csv}.Invoke()
foreach($item in (Get-PnPListItem -list $LabelActivitiesList -PageSize 1000 -Query $CAML)){
    $target=-1
    for($i=0;$i -lt $csv2.count;$i++){
        if($csv2[$i].LogId -eq $item.FieldValues["Title"]){
            $target=$i
            continue
        }
    }
    if($target -ge 0){
        $csv2.RemoveAt($target)
    }
}
"Newly identified total labeling activities since $Start"+": "+$csv2.count

#リスト側にない検索結果はリストアイテムとして新規に登録する
#Add-PnPListItem の Batch 処理だと UTC でタイムスタンプが書き込めなかったため CSOM を利用
$ctx=get-pnpcontext
$list = $ctx.get_web().get_lists().getByTitle($LabelActivitiesList)
$count=0
foreach($item in $csv2){
    $lic = New-Object Microsoft.SharePoint.Client.ListItemCreationInformation
    $i = $list.AddItem($lic)
    $i.set_item("Title", $item.LogId)
    $i.set_item("User", $item.User)
    $i.set_item("Time", $item.Time)
    $i.set_item("Item", $item.Item)
    $i.set_item("Operation", $item.Operation)
    $i.set_item("CurrentLabel", $item.CurrentLabel)
    $i.set_item("PreviousLabel", $item.PreviousLabel)
    $i.set_item("EncryptionStatusChange", $item.EncryptionStatusChange)
    $i.set_item("Justification", $item.Justification)
    $i.set_item("Workload", $item.Workload)
    $i.update()
    $count++
    #書き込みが多い場合には、一旦 100 アイテムで反映
    if($count % 100 -eq 0){
        $ctx.ExecuteQuery()}
    }
$ctx.ExecuteQuery()
"Label activities were synched with the list."
Disconnect-PnPOnline
```
## 3. Power Automate によるメール通知の設定
[こちら](https://github.com/YoshihiroIchinose/E5Comp/edit/main/ExternalSharingMonitoring.md)を参考に、LabelActivities のリストに対して、
新規リスト アイテムが登録された際、起動する Power Automate のフローを作成し、カスタムのメール通知を設定する。
