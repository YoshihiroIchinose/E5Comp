# Endpoint DLP の操作をメール通知する
Endpoint DLP で監視できる、クラウドへのファイルのアップロードおよびリムーバブル メディアへの書き込み操作を、Azure Automate 上に構成した PowerShell スクリプトを通じて、Microsoft 365 監査ログより取得し、
SharePoint 重複を排除してリストに書き込むと共に SharePoint リストへの新規アイテム追加をトリガーとした Power Automate を利用して、カスタムのメール通知を行うソリューションの実装サンプルになります。
なお、このサンプルでは、FileCopiedToRemovableMedia および FileUploadedToCloud のログを対象とした通知を行います。
構造や作り方は外部共有操作をメール通知する[こちら](https://github.com/YoshihiroIchinose/E5Comp/blob/main/ExternalSharingMonitoring.md)のサンプルと同じなので、
併せて参考にしてください。

おおよその動作の仕組みとしては以下の通りです。
1. Endpoint DLP の操作を記録する SharePoint リストを用意しておく
2. 上記 SharePoint リストに Power Automate で新しいアイテムが追加された際、操作を行ったユーザーにメール通知を行う処理を設定しておく
3. Azure Automate を用いて、一定期間ごとに監査ログから特定のラベル操作を抜き出し、重複を排除しながらログを 1 のリストに書き込む

考慮事項
1. Azure Automate では高額ではないが処理量に応じた従量課金が発生
2. Power Automate のメール通知では、設定を行ったユーザーが送信者となったメール通知となり、Office 365 組み込みのライセンスでは、1 日 6,000 件といった処理量の上限あり
3. 重複を排除したラベル操作の追記をリストに行うため、Auzre Automate で抜き出すログの範囲は重複してもよい。もし数時間単位でタイムリーに通知を行いたい場合、Azure Automate で抜き出すログの範囲も直近数時間といった範囲に狭め、スケジュール実行の頻度を数時間サイクルとすること。
4. 新規作成されたリスト アイテムに紐づく Power Automate のフローを用いるため、1 操作 = 1 メール通知となり、複数操作をサマリしたメール通知を行うことはできない
5. テスト用に再度メール通知を行いたい場合には、SharePoint のリストから、該当の操作のログを消すこと。これにより Azure Automate で再度ログが登録しなおされた際、メール通知が Power Automate により行われる。

# 事前準備
## 1. EndpintDLP 操作を書き出す SharePoint サイトの準備
1. 専用の SharePoint チーム サイトを CustomeNotifcation の名称で作成する   
1. 作成したサイトのサイト コンテンツから EndpointDLP の名称で空白のリストを作成する   
### EndpointDLP
タイトル以外の以下の列を追加する。PowerShell で扱いやすくするために、以下の通り英数字で列は作成する。   
もし列名をミスタイプした場合には、列名を変更して修正するだけでは列の内部名は修正されないので、列を削除した上で、作り直す。
| 列の種類 | 列の名称 |
|---|---|
| 1 行テキスト | User, Itme, Operation, EnforcementMode, TargetFilePath, OriginatingDomain, TargetDomain, PolicyName, RuleName, ClientIP, DeviceName, Application, Sha1, Sha256, FileType, EvidenceFile,RemovableMediaDeviceAttributes|
| 日付と時刻 (時刻を含める) *| Time |
| はい/いいえ | RMSEncrypted, JitTriggered,Notified|
| 数値 | FileSize |
| 複数行テキスト | SensitiveInfoTypeData|

*日付と時刻の種類を選び、時刻を含めるをはいとする。また LabelActivities のリストの設定から、列のインデックス付きの列で、Time にインデックスを設定しておく。   
こちらのリストにログが書き込まれると以下のような見た目となる。   
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/EDLPNotif01.png"/>

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
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/EDLPNotif02.png"/>

#### Aure Automation サンプル スクリプト

```
$Credential = Get-AutomationPSCredential -Name "Office 365"
$SiteUrl="https://xxxx.sharepoint.com/sites/CustomNotification"
$LabelActivitiesList="EndpointDLP"
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

#機密情報の情報を複数行テキストに変換する Function
Function PurseSIT{
    Param($a)
    $b=ConvertFrom-Json $a
    $output=""
    foreach($i in $b){
        $detail=@()
        foreach($d in $i.SensitiveInformationDetailedClassificationAttributes){
            $detail+=$d.Confidence.ToString() +"x"+$d.Count
        }
        $line=[string]::Join(",", $detail)
        $output+=$i.SensitiveInfoTypeName+":"+$i.Confidence+"x"+$i.Count+" ("+$line+")"+"`n"
    }
    return $output
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
enum EnforcementMode {
    None = 0
    Audit = 1
    Warn = 2
    WarnAndBypass = 3 
    Block = 4
}
#3 $global:output から DLP 操作の内容を抽出し、$global:csv に書き出す
Function FormatLabelActitivyLog {
    foreach($i in $global:output){
       $AuditData=$i.AuditData|ConvertFrom-Json
        $line = New-Object -TypeName PSObject
        
        #列名が違うか加工が必要な属性の処理
        AddMember $line "LogId" $Auditdata.Id
        AddMember $line "User" $i.UserIds
        AddMember $line "Time" $i.CreationDate.ToString("yyyy-MM-ddTHH:mm:ssZ")
        AddMember $line "Operation" $i.Operations
        AddMember $line "Item" $AuditData.ObjectId
        AddMember $line "EnforcementMode" ([EnforcementMode].GetEnumName($Auditdata.EnforcementMode))
        AddMember $line "SensitiveInfoTypeData" (PurseSIT (ConvertTo-Json -Compress -Depth 10 $AuditData.SensitiveInfoTypeData))
        AddMember $line "EvidenceFile" $AuditData.EvidenceFile.FullUrl
        AddMember $line "RemovableMediaDeviceAttributes" $AuditData.RemovableMediaDeviceAttributes
        AddMember $line "PolicyName" $AuditData.PolicyMatchInfo.PolicyName
        AddMember $line "RuleName" $AuditData.PolicyMatchInfo.RuleName

        #ログ内の列名と、書き込む先の列名が一致し、加工が必要ない列
        $att=@("ClientIP","DeviceName","Application","Sha1","Sha256","FileSize","RMSEncrypted","TargetFilePath",
        "OriginatingDomain","TargetDomain","JitTriggered","FileType")

        foreach($a in $att){
            AddMember $line $a  ($AuditData.$a)
        }
        $global:csv+=$line
    }
}

#Exchange Online に接続
Connect-ExchangeOnline -credential $credential
$csv=@()

#DLPEndpoint のログ取得
$output=@()
ExtractAuditLog "DLPEndpoint" "FileCopiedToRemovableMedia,FileUploadedToCloud" 
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
"Newly identified total EndpointDLP activities since $Start"+": "+$csv2.count

#リスト側にない検索結果はリストアイテムとして新規に登録する
#Add-PnPListItem の Batch 処理だと UTC でタイムスタンプが書き込めなかったため CSOM を利用
$ctx=get-pnpcontext
$list = $ctx.get_web().get_lists().getByTitle($LabelActivitiesList)
$count=0
foreach($item in $csv2){
    $lic = New-Object Microsoft.SharePoint.Client.ListItemCreationInformation
    $i = $list.AddItem($lic)
    $i.set_item("Title", $item.LogId)

    #列名が一致するものは、そのまま書き込み
    $att=@("User","Time","Operation","Item","EnforcementMode","PolicyName","RuleName","ClientIP",
    "DeviceName","Application","FileSize","SensitiveInfoTypeData","RMSEncrypted","TargetFilePath",
    "RemovableMediaDeviceAttributes","OriginatingDomain","TargetDomain","JitTriggered", 
    "FileType", "EvidenceFile")

    foreach($a in $att){
        $i.set_item($a,$item.$a)
    }

    #数字が入る列名は SharePoint では、特殊な列名に変換されているため、その内部列の名称に合わせて書き込み
    $i.set_item("_x0053_ha1", $item.Sha1)
    $i.set_item("_x0053_ha256", $item.Sha256)

    $i.update()
    $count++
    #書き込みが多い場合には、一旦 100 アイテムで反映
    if($count % 100 -eq 0){
        $ctx.ExecuteQuery()}
    }
$ctx.ExecuteQuery()
"Endpoint DLP activities were synched with the list."
Disconnect-PnPOnline
```
## 3. Power Automate によるメール通知の設定
[こちら](https://github.com/YoshihiroIchinose/E5Comp/blob/main/ExternalSharingMonitoring.md)を参考に、LabelActivities のリストに対して、
新規リスト アイテムが登録された際、起動する Power Automate のフローを作成し、カスタムのメール通知を設定する。必要に応じて EncryptionStatusChange で、EncryptionRemoved のみを対象としてもよい。
