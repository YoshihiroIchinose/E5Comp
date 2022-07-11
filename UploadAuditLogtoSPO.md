# Azure Automation を利用して Office 365 の監査ログから特定の操作を CSV 形式で出力し SharePoint Online サイトにアップロードする
こちらのサンプルでは、DLPEndpoint のログの種類から、"FileAccessedByUnallowedApp", "FileCopiedToRemovableMedia", "FilePrinted","FileUploadedToCloud" の 4 つのログを、
最大 5 万件、昨日から 32 日前までの 31 日分のログを抽出し、 CSV 形式で SharePoint Online サイトにアップロードします。
なお Audit Data に含まれる、ログについては、1 階層分のみパースして、CSV の列としています。
またこのスクリプトにより、固定の名前の CSV ファイルとして SharePoint Online 上に出力し、 Daily 更新することができるので、SharePoint Online 上の CSV ファイルを
データソースとして、Power BI に取り込んで、Daily 更新のダッシュボードを作ることも可能となります。

## 準備
1. Azure 環境にて Azure Automation アカウントを作成
2. Azure Automation アカウントにて、"モジュール" -> "ギャラリーを参照"から、以下の 3 つのモジュールを追加する。   
(ランタイム バージョンは 5.1)   
  SharePointOnline.CSOM    
  Microsoft.Online.SharePoint.PowerShell   
  ExchangeOnlineManagement   
3. Azure Automation アカウントの"資格情報"->"資格情報の追加"で、監査ログの抽出権限があり、   
  指定の SharePoint Online サイトに投稿権限があるアカウントの ID とパスワードを登録しておく。
4. Azure Automation アカウントの"Runbook"->"Runbook の作成"で PowerShell、ランタイム バージョンの 5.1 の Runbook を作成する
5. 作成した Runbook に以下のスクリプトをコピー ペストする
6. 適宜スクリプト内の SharePoint Site の URL および、ファイル保存先の相対 URL (FQDN を除いたもの) を書き変え、保存し、公開する
7. 作成した Runbook を"開始"し、動作を確認する
8. 必要に応じて Daily 等のスケジュール実行を設定する

## スクリプト
```
#変数
$date=Get-Date
$Startdate=$date.addDays(-32).ToString("yyyy/MM/dd")
$Enddate=$date.addDays(-1).ToString("yyyy/MM/dd")
$outfile="C:\Report\DLPEndpoint.csv"
$siteUrl="https://xxxx.sharepoint.com/sites/Contents2/"
$targeturl ="/sites/DLPLogs/Shared Documents/DLPEndpoint.csv"

#対象のログ
$RecordType="DLPEndpoint"
$Operations="FileAccessedByUnallowedApp","FileCopiedToRemovableMedia","FilePrinted","FileUploadedToCloud"

#その他のログ
#"ArchiveCreated","FileCopiedToClipboard","FileCopiedToRemoteDesktopSession","FileCreated",
#"FileCreatedOnNetworkShare","FileCreatedOnRemovableMedia","FileDeleted","FileDownloadedFromBrowser",
#"FileModified","FileRead","FileRenamed","RemovableMediaMount","RemovableMediaUnmount"

#Credentialの生成
$Credential = Get-AutomationPSCredential -Name "Office 365"

Import-Module ExchangeOnlineManagement
Connect-ExchangeOnline -credential $Credential

#日付と時刻で固有のセッション ID 文字列を生成
$sessionId=$RecordType+(Get-Date -Format "yyyyMMdd-HHmm")

#最大 5,000 x 10 回のループでログを取得
$output=@();
for($i = 0; $i -lt 10; $i++){
    $result=Search-UnifiedAuditLog -RecordType $RecordType -Operations $Operations -Startdate $Startdate -Enddate $Enddate -SessionId $sessionId -SessionCommand ReturnLargeSet -ResultSize 5000
    $output+=$result
    "Query "+($i+1)+" round: "+$result.Count.ToString() + " results"
    if($result.count -ne 5000){break}
}
"Total: "+$output.Count.ToString() + " results"
    
#Operation の種類ごとに最初の 1 つ目のアイテムから Json に含まれているフィールドを取得
$OperationTypes=$output|Group-Object Operations
$FieldName=@()
foreach($Operation in $OperationTypes){
    $JsonRaw=$Operation.Group[0].AuditData|ConvertFrom-Json
    $FieldsInJson=$JsonRaw|get-member -type NoteProperty
    foreach($f in $FieldsInJson){
      if($FieldName.Contains($f.Name) -eq $false) { $FieldName+=$f.Name}
    }
}

#Select-Object で利用するために、Json をパースする ScriptBlock を生成
$Fields="ResultIndex", "CreationDate","UserIds","Operations","RecordType"
foreach($f in $FieldName){
    $sb1=[scriptblock]::Create('$JsonRaw.'+$f)
    $sb2=[scriptblock]::Create('$att=$JsonRaw.'+$f+';if($att.GetType().Name -eq "Object[]" -or $att.GetType().Name -eq "PSCustomObject"){ConvertTo-Json -Compress -Depth 10 $att} else {$att}')
    if($f -ne "RecordType") {$Fields+=@{Name=$f;Expression=$sb2}}
    else {$Fields+=@{Name="RecordType2";Expression=$sb1}}
}

#Jsonをパースしながら、CSV 形式に加工
$csv=@();
foreach($row in $output){
    $JsonRaw=$row.AuditData|ConvertFrom-Json
    $data=$row|Select-Object -Property $Fields
    $csv+=$data
 }

#出力
$csv|Export-Csv -Path $outfile -NoTypeInformation -Encoding UTF8

#CSOM のアセンブリのロード
Load-SPOnlineCSOMAssemblies
$ctx = New-Object Microsoft.SharePoint.Client.ClientContext($siteUrl)
$username =$Credential.UserName
$password = $Credential.Password
$cre=$null
$count=0
while($cre -eq $null -and $count -lt 10){
$count++
try{$cre = New-Object Microsoft.SharePoint.Client.SharePointOnlineCredentials($username,$password)}
catch{
    "Authentication Error!"
    $_.Exception.Message
    Start-Sleep -s 5}
}
$ctx.Credentials=$cre

$fs = new-object System.IO.FileStream($outfile ,[System.IO.FileMode]::Open,[System.IO.FileAccess]::Read)
[Microsoft.SharePoint.Client.File]::SaveBinaryDirect($ctx,$targeturl , $fs, $true)
$fs.Close()
"Upload completed."
```
