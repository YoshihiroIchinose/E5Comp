# Azure Automation を利用して Office 365 の監査ログから特定の操作を CSV 形式で出力し SharePoint Online サイトにアップロードする
こちらのサンプルでは、DLPEndpoint のログの種類から、"FileAccessedByUnallowedApp", "FileCopiedToRemovableMedia", "FilePrinted","FileUploadedToCloud" の 4 つのログを、
最大 5 万件、昨日から 31 日前までの 31 日分の範囲でログを抽出し、 CSV 形式で SharePoint Online サイトにアップロードします。
なお Audit Data に含まれる、ログについては、1 階層分のみパースして、CSV の列としています。
またこのスクリプトにより、固定の名前の CSV ファイルとして SharePoint Online 上に出力し、 Daily 更新することができるので、SharePoint Online 上の CSV ファイルを
データソースとして、Power BI に取り込んで、Daily 更新のダッシュボードを作ることも可能となります。なおこちらのスクリプトをベースに、$RecordType="SensitivityLabelAction"、$Operations="" とすることで、秘密度ラベルの付与・変更・削除の操作のログを同様に CSV 形式で出力し、特定の SharePoint Online サイトにアップロード可能です。

## 準備
1. Azure 環境にて Azure Automation アカウントを作成
2. Azure Automation アカウントにて、"モジュール" -> "ギャラリーを参照"から、以下の 3 つのモジュールを追加する。   
(ランタイム バージョンは 5.1)   
  SharePointOnline.CSOM    
  Microsoft.Online.SharePoint.PowerShell   
  ExchangeOnlineManagement   
3. Azure Automation アカウントの"資格情報"->"資格情報の追加"で、監査ログの抽出権限があり、   
  指定の SharePoint Online サイトに投稿権限があるアカウントの ID とパスワードを "Office 365" という名称で登録しておく。
4. Azure Automation アカウントの"Runbook"->"Runbook の作成"で PowerShell、ランタイム バージョンの 5.1 の Runbook を作成する
5. 作成した Runbook に以下のスクリプトをコピー & ペーストする
6. 適宜スクリプト内の SharePoint Site の URL および、ファイル保存先の相対 URL (FQDN を除いたもの) を書き変え、保存し、公開する
7. 作成した Runbook を"開始"し、動作を確認する
8. 必要に応じて Daily 等のスケジュール実行を設定する

## RecordType と Operations を指定して監査ログを SPO にアップロードする Azure Automation の Powershell スクリプト
```
#変数
$date=Get-Date
$Startdate=$date.addDays(-31).ToString("yyyy/MM/dd")
$Enddate=$date.ToString("yyyy/MM/dd")
$RecordType="DLPEndpoint"
$outfile="C:\Report\"+$RecordType+".csv"
$siteUrl="https://xxx.sharepoint.com/sites/DLPLogs/"
$targeturl ="/sites/DLPLogs/Shared Documents/"+$RecordType+".csv"

#取得対象の Operataions (指定なしも可能)
$Operations="FileAccessedByUnallowedApp","FileCopiedToRemovableMedia","FilePrinted","FileUploadedToCloud"

#その他の Operations
#"ArchiveCreated","FileCopiedToClipboard","FileCopiedToRemoteDesktopSession","FileCreated",
#"FileCreatedOnNetworkShare","FileCreatedOnRemovableMedia","FileDeleted","FileDownloadedFromBrowser",
#"FileModified","FileRead","FileRenamed","RemovableMediaMount","RemovableMediaUnmount"

#Credential の生成と接続
$Credential = Get-AutomationPSCredential -Name "Office 365"
Connect-ExchangeOnline -credential $Credential

#日付と時刻で固有のセッション ID 文字列を生成
$sessionId=$RecordType+(Get-Date -Format "yyyyMMdd-HHmm")

#最大 5,000 x 10 回のループでログを取得
$output=@();
for($i = 0; $i -lt 10; $i++){
	if($Operations -ne $null -and $Operations.Length -ne 0){
    		$result=Search-UnifiedAuditLog -RecordType $RecordType -Operations $Operations -Startdate $Startdate -Enddate $Enddate -SessionId $sessionId -SessionCommand ReturnLargeSet -ResultSize 5000
	}
	else {
		$result=Search-UnifiedAuditLog -RecordType $RecordType -Startdate $Startdate -Enddate $Enddate -SessionId $sessionId -SessionCommand ReturnLargeSet -ResultSize 5000
	}
    $output+=$result
    "Query "+($i+1)+" round: "+$result.Count.ToString() + " results"
    if($result.count -ne 5000){break}
}
Disconnect-ExchangeOnline -Confirm:$false
if($output.count -eq 0){
	"No data"
	exit
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

#CSOM のアセンブリのロードと SPO への接続
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
$ctx.Dispose()
"Upload completed."
```
## Power BI レポート サンプル
SharePoint Online サイト上の CSV ファイルは、Web 版の Power BI のいずれかのワークスペースから、"データセットの追加"で、"ファイル"を"取得"を選択して、レポートに取り込むことが可能で、1 時間ごとに更新されたデータを自動取得できる。
![サンプル](img/PowerBI_EDLP.png)

## 2022/07/13 時点で全種類の監査ログを SPO にアップロードする Azure Automation の Powershell スクリプト(検証用)
```
#変数
$global:date=Get-Date
$global:Startdate=$date.addDays(-31).ToString("yyyy/MM/dd")
$global:Enddate=$date.ToString("yyyy/MM/dd")

#対象のログ
$RecordTypes="ExchangeAdmin","ExchangeItem","ExchangeItemGroup","SharePoint","SyntheticProbe","SharePointFileOperation"
$RecordTypes+="OneDrive","AzureActiveDirectory","AzureActiveDirectoryAccountLogon","DataCenterSecurityCmdlet","ComplianceDLPSharePoint"
$RecordTypes+="Sway","ComplianceDLPExchange","SharePointSharingOperation","AzureActiveDirectoryStsLogon","SkypeForBusinessPSTNUsage"
$RecordTypes+="SkypeForBusinessUsersBlocked","SecurityComplianceCenterEOPCmdlet","ExchangeAggregatedOperation","PowerBIAudit"
$RecordTypes+="CRM","Yammer","SkypeForBusinessCmdlets","Discovery","MicrosoftTeams","ThreatIntelligence","MailSubmission"
$RecordTypes+="MicrosoftFlow","AeD","MicrosoftStream","ComplianceDLPSharePointClassification","ThreatFinder","Project"
$RecordTypes+="SharePointListOperation","SharePointCommentOperation","DataGovernance","Kaizala","SecurityComplianceAlerts"
$RecordTypes+="ThreatIntelligenceUrl","SecurityComplianceInsights","MIPLabel","WorkplaceAnalytics","PowerAppsApp","PowerAppsPlan"
$RecordTypes+="ThreatIntelligenceAtpContent","LabelContentExplorer","TeamsHealthcare","ExchangeItemAggregated","HygieneEvent"
$RecordTypes+="DataInsightsRestApiAudit","InformationBarrierPolicyApplication","SharePointListItemOperation","SharePointContentTypeOperation"
$RecordTypes+="SharePointFieldOperation","MicrosoftTeamsAdmin","HRSignal","MicrosoftTeamsDevice","MicrosoftTeamsAnalytics"
$RecordTypes+="InformationWorkerProtection","Campaign","DLPEndpoint","AirInvestigation","Quarantine","MicrosoftForms","ApplicationAudit"
$RecordTypes+="ComplianceSupervisionExchange","CustomerKeyServiceEncryption","OfficeNative","MipAutoLabelSharePointItem","MipAutoLabelSharePointPolicyLocation"
$RecordTypes+="MicrosoftTeamsShifts","SecureScore","MipAutoLabelExchangeItem","CortanaBriefing","Search","WDATPAlerts","PowerPlatformAdminDlp"
$RecordTypes+="PowerPlatformAdminEnvironment","MDATPAudit","SensitivityLabelPolicyMatch","SensitivityLabelAction","SensitivityLabeledFileAction"
$RecordTypes+="AttackSim","AirManualInvestigation","SecurityComplianceRBAC","UserTraining","AirAdminActionInvestigation","MSTIC","PhysicalBadgingSignal"
$RecordTypes+="TeamsEasyApprovals","AipDiscover","AipSensitivityLabelAction","AipProtectionAction","AipFileDeleted","AipHeartBeat","MCASAlerts"
$RecordTypes+="OnPremisesFileShareScannerDlp","OnPremisesSharePointScannerDlp","ExchangeSearch","SharePointSearch","PrivacyDataMinimization"
$RecordTypes+="LabelAnalyticsAggregate","MyAnalyticsSettings","SecurityComplianceUserChange","ComplianceDLPExchangeClassification","ComplianceDLPEndpoint"
$RecordTypes+="MipExactDataMatch","MSDEResponseActions","MSDEGeneralSettings","MSDEIndicatorsSettings","MS365DCustomDetection","MSDERolesSettings"
$RecordTypes+="MAPGAlerts","MAPGPolicy","MAPGRemediation","PrivacyRemediationAction","PrivacyDigestEmail","MipAutoLabelSimulationProgress"
$RecordTypes+="MipAutoLabelSimulationCompletion","MipAutoLabelProgressFeedback","DlpSensitiveInformationType","MipAutoLabelSimulationStatistics"
$RecordTypes+="LargeContentMetadata","Microsoft365Group","CDPMlInferencingResult","FilteringMailMetadata","CDPClassificationMailItem"
$RecordTypes+="CDPClassificationDocument","OfficeScriptsRunAction","FilteringPostMailDeliveryAction","CDPUnifiedFeedback","TenantAllowBlockList"
$RecordTypes+="ConsumptionResource","HealthcareSignal","DlpImportResult","CDPCompliancePolicyExecution","MultiStageDisposition","PrivacyDataMatch"
$RecordTypes+="FilteringDocMetadata","FilteringEmailFeatures","PowerBIDlp","FilteringUrlInfo","FilteringAttachmentInfo","CoreReportingSettings"
$RecordTypes+="ComplianceConnector","PowerPlatformLockboxResourceAccessRequest","PowerPlatformLockboxResourceCommand","CDPPredictiveCodingLabel"
$RecordTypes+="CDPCompliancePolicyUserFeedback","WebpageActivityEndpoint","OMEPortal","CMImprovementActionChange","FilteringUrlClick"
$RecordTypes+="MipLabelAnalyticsAuditRecord","FilteringEntityEvent","FilteringRuleHits","FilteringMailSubmission","LabelExplorer","MicrosoftManagedServicePlatform"
$RecordTypes+="PowerPlatformServiceActivity","ScorePlatformGenericAuditRecord","FilteringTimeTravelDocMetadata","Alert","AlertStatus"
$RecordTypes+="AlertIncident","IncidentStatus","Case","CaseInvestigation","RecordsManagement","PrivacyRemediation","DataShareOperation"
$RecordTypes+="CdpDlpSensitive","EHRConnector","FilteringMailGradingResult","PublicFolder","PrivacyTenantAuditHistoryRecord","AipScannerDiscoverEvent"
$RecordTypes+="EduDataLakeDownloadOperation","M365ComplianceConnector","MicrosoftGraphDataConnectOperation"

#対象の SharePoint Online のサイトおよびパス (事前にドキュメント ライブラリ・フォルダを作成のこと)
$global:siteUrl="https://xxxx.sharepoint.com/sites/DLPLogs/"
$global:targeturl ="/sites/DLPLogs/Shared Documents/Logs/"

#Credentialの生成
$global:Credential = Get-AutomationPSCredential -Name "Office 365"

function GetLogandUpload{
Param (
		[string]$RecordType, [string]$Operations=""
    )
"Get logs: "+$RecordType
#日付と時刻で固有のセッション ID 文字列を生成
$sessionId=$RecordType+(Get-Date -Format "yyyyMMdd-HHmm")

#最大 5,000 x 10 回のループでログを取得
$output=@();
for($i = 0; $i -lt 10; $i++){
	if($Operations -ne $null -and $Operations.Length -ne 0)
	{
    	$result=Search-UnifiedAuditLog -RecordType $RecordType -Operations $Operations -Startdate $Startdate -Enddate $Enddate -SessionId $sessionId -SessionCommand ReturnLargeSet -ResultSize 5000
	}
	else
	{
		$result=Search-UnifiedAuditLog -RecordType $RecordType -Startdate $Startdate -Enddate $Enddate -SessionId $sessionId -SessionCommand ReturnLargeSet -ResultSize 5000
	}
    $output+=$result
    "Query "+($i+1)+" round: "+$result.Count.ToString() + " results"
    if($result.count -ne 5000){break}
}
if($output.count -eq 0){
	"No data"
	return
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
$outfile="C:\Report\"+$RecordType+".csv"
$csv|Export-Csv -Path $outfile -NoTypeInformation -Encoding UTF8

$fs = new-object System.IO.FileStream($outfile ,[System.IO.FileMode]::Open,[System.IO.FileAccess]::Read)
[Microsoft.SharePoint.Client.File]::SaveBinaryDirect($global:ctx,$global:targeturl+$RecordType+".csv" , $fs, $true)
$fs.Close()
"Upload completed."
}

#メイン処理
Connect-ExchangeOnline -credential $global:Credential

#CSOM のアセンブリのロード
Load-SPOnlineCSOMAssemblies
$global:ctx = New-Object Microsoft.SharePoint.Client.ClientContext($global:siteUrl)
$cre=$null
$count=0
while($cre -eq $null -and $count -lt 10){
$count++
try{$cre = New-Object Microsoft.SharePoint.Client.SharePointOnlineCredentials($Credential.UserName,$Credential.Password)}
catch{
    "SPO Authentication Error"
    $_.Exception.Message
    Start-Sleep -s 5}
}
$global:ctx.Credentials=$cre

#Do Loop
foreach($r in $RecordTypes){
	GetLogandUpload($r)
}

#Close
$ctx.Dispose()
Disconnect-ExchangeOnline -Confirm:$false
```
