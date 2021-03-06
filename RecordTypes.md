# Office 365 監査ログでの RecordType について
[Docs](https://docs.microsoft.com/en-us/office/office-365-management-api/office-365-management-activity-api-schema#auditlogrecordtype) に AuditRecordType の取りうる値が一覧で公開されていますが、公開されているものだけが全部ではなく、更新が反映されていないものあります。内部的に定義された RecordTypes の文字列については、Search-UnifiedAuditLog の -RecrodType の引数のエラーから取得することができます。各 RecordTypes の文字列に振られている列挙体としての値については、New-UnifiedAuditLogRetentionPolicy の -RecordTypes の引数に文字列として、RecordTypes を指定し、その後 Set-UnifiedAuditLogRetentionPolicy で得られた設定の中で、 RecordTypes の値がどのような数字になっているかで判断できます。2022 年 4 月時点で、いくつかの欠番はありますが 1 から 171 までの値があります。なお Advanced Audit の機能で、有効なライセンスがあれば、自動的に 90 日保持されるログの種類については、こちらの [Docs](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/audit-log-retention-policies?view=o365-worldwide#more-information)
 に記載があります。

# Office 365 監査ログでの RecordType 一覧
| 番号 | 名前 | Docs への記載 | Advanced Audit で既定で 1 年間の保持 | UI からの保持ポリシー設定対象 | Sentinel 対応(仮) |
|:---|:---------|:----:|:----:|:----:|:----:|
|1|ExchangeAdmin|Yes|Yes|Yes|Yes|Yes|
|2|ExchangeItem|Yes|Yes|Yes|
|3|ExchangeItemGroup|Yes|Yes|Yes|
|4|SharePoint|Yes|Yes|Yes|Yes
|5|SyntheticProbe||||
|6|SharePointFileOperation|Yes|Yes|Yes|Yes
|7|OneDrive|Yes|||
|8|AzureActiveDirectory|Yes|Yes|Yes|
|9|AzureActiveDirectoryAccountLogon|Yes|Yes||
|10|DataCenterSecurityCmdlet|Yes|||
|11|ComplianceDLPSharePoint|Yes|Yes||
|12|Sway|||Yes|
|13|ComplianceDLPExchange|Yes|Yes||
|14|SharePointSharingOperation|Yes|Yes|Yes|
|15|AzureActiveDirectoryStsLogon|Yes|Yes||
|16|SkypeForBusinessPSTNUsage|Yes|||
|17|SkypeForBusinessUsersBlocked|Yes|||
|18|SecurityComplianceCenterEOPCmdlet|Yes||Yes|
|19|ExchangeAggregatedOperation|Yes|Yes||
|20|PowerBIAudit|Yes||Yes|
|21|CRM|Yes||Yes|
|22|Yammer|Yes||Yes|
|23|SkypeForBusinessCmdlets|Yes||Yes|
|24|Discovery|Yes||Yes|
|25|MicrosoftTeams|Yes||Yes|Yes
|26|欠番||||
|27|欠番||||
|28|ThreatIntelligence|Yes|||
|29|MailSubmission|Yes|||
|30|MicrosoftFlow|Yes||Yes|
|31|AeD|Yes||Yes|
|32|MicrosoftStream|Yes||Yes|
|33|ComplianceDLPSharePointClassification|Yes|Yes||
|34|ThreatFinder|Yes|||
|35|Project|Yes|Yes||
|36|SharePointListOperation|Yes|Yes|Yes|Yes
|37|SharePointCommentOperation|Yes|Yes||
|38|DataGovernance|Yes||Yes|
|39|Kaizala|Yes|||
|40|SecurityComplianceAlerts|Yes|||
|41|ThreatIntelligenceUrl|Yes|||
|42|SecurityComplianceInsights|Yes|||
|43|MIPLabel|Yes|||
|44|WorkplaceAnalytics|Yes||Yes|
|45|PowerAppsApp|Yes||Yes|
|46|PowerAppsPlan|Yes|||
|47|ThreatIntelligenceAtpContent|Yes|||
|48|LabelContentExplorer|Yes||Yes|
|49|TeamsHealthcare|Yes|||
|50|ExchangeItemAggregated|Yes|Yes|Yes|Yes
|51|HygieneEvent|Yes|||
|52|DataInsightsRestApiAudit|Yes|||
|53|InformationBarrierPolicyApplication|Yes|Yes||
|54|SharePointListItemOperation|Yes|||
|55|SharePointContentTypeOperation|Yes|Yes|Yes|
|56|SharePointFieldOperation|Yes|Yes|Yes|
|57|MicrosoftTeamsAdmin|Yes||Yes||
|58|HRSignal|Yes|||
|59|MicrosoftTeamsDevice|Yes|||
|60|MicrosoftTeamsAnalytics|Yes|||
|61|InformationWorkerProtection|Yes|||
|62|Campaign|Yes|Yes||
|63|DLPEndpoint|Yes||Yes|
|64|AirInvestigation|Yes|||
|65|Quarantine|Yes||Yes|
|66|MicrosoftForms|Yes||Yes|
|67|ApplicationAudit|Yes|||
|68|ComplianceSupervisionExchange|Yes|Yes||Yes
|69|CustomerKeyServiceEncryption|Yes|Yes|Yes|
|70|OfficeNative|Yes|||
|71|MipAutoLabelSharePointItem|Yes|||
|72|MipAutoLabelSharePointPolicyLocation|Yes|||
|73|MicrosoftTeamsShifts|Yes|||
|74|SecureScore||||
|75|MipAutoLabelExchangeItem|Yes|||
|76|CortanaBriefing|Yes|||
|77|Search|Yes|||
|78|WDATPAlerts|Yes|||
|79|PowerPlatformAdminDlp||||
|80|PowerPlatformAdminEnvironment||||
|81|MDATPAudit|||Yes|
|82|SensitivityLabelPolicyMatch|Yes|||
|83|SensitivityLabelAction|Yes|||
|84|SensitivityLabeledFileAction|Yes|||
|85|AttackSim|Yes|||
|86|AirManualInvestigation|Yes|||
|87|SecurityComplianceRBAC|Yes|||
|88|UserTraining|Yes|||
|89|AirAdminActionInvestigation|Yes|||
|90|MSTIC|Yes||Yes|
|91|PhysicalBadgingSignal|Yes|||
|92|TeamsEasyApprovals|||Yes|
|93|AipDiscover|Yes||Yes|
|94|AipSensitivityLabelAction|Yes||Yes|
|95|AipProtectionAction|Yes||Yes|
|96|AipFileDeleted|Yes||Yes|
|97|AipHeartBeat|Yes||Yes|
|98|MCASAlerts|Yes|||
|99|OnPremisesFileShareScannerDlp|Yes||Yes|
|100|OnPremisesSharePointScannerDlp|Yes||Yes|
|101|ExchangeSearch|Yes||Yes|
|102|SharePointSearch|Yes||Yes|
|103|PrivacyDataMinimization|Yes|||
|104|LabelAnalyticsAggregate||||
|105|MyAnalyticsSettings|Yes||Yes|
|106|SecurityComplianceUserChange|Yes|||
|107|ComplianceDLPExchangeClassification|Yes|||
|108|ComplianceDLPEndpoint||||
|109|MipExactDataMatch|Yes||Yes|
|110|MSDEResponseActions|||Yes|
|111|MSDEGeneralSettings|||Yes|
|112|MSDEIndicatorsSettings|||Yes|
|113|MS365DCustomDetection|Yes||Yes|
|114|MSDERolesSettings|||Yes|
|115|MAPGAlerts|||Yes|
|116|MAPGPolicy|||Yes|
|117|MAPGRemediation|||Yes|
|118|PrivacyRemediationAction||||
|119|PrivacyDigestEmail||||
|120|MipAutoLabelSimulationProgress||||
|121|MipAutoLabelSimulationCompletion||||
|122|MipAutoLabelProgressFeedback||||
|123|DlpSensitiveInformationType|||Yes|
|124|MipAutoLabelSimulationStatistics||||
|125|LargeContentMetadata||||
|126|Microsoft365Group|||Yes|
|127|CDPMlInferencingResult||||
|128|FilteringMailMetadata||||
|129|CDPClassificationMailItem||||
|130|CDPClassificationDocument||||
|131|OfficeScriptsRunAction|||Yes||
|132|FilteringPostMailDeliveryAction||||
|133|CDPUnifiedFeedback||||
|134|TenantAllowBlockList||||
|135|ConsumptionResource||||
|136|HealthcareSignal||||
|137|欠番||||
|138|DlpImportResult|||Yes|
|139|CDPCompliancePolicyExecution
|140|MultiStageDisposition|||Yes|
|141|PrivacyDataMatch||||
|142|FilteringDocMetadata||||
|143|FilteringAttachmentInfo||||
|144|PowerBIDlp||||
|145|FilteringUrlInfo||||
|146|FilteringAttachmentInfo||||
|147|CoreReportingSettings|Yes|Yes||
|148|ComplianceConnector||||
|149|PowerPlatformLockboxResourceAccessRequest|||Yes||
|150|PowerPlatformLockboxResourceCommand|||Yes||
|151|CDPPredictiveCodingLabel||||
|152|CDPCompliancePolicyUserFeedback||||
|153|WebpageActivityEndpoint||||
|154|OMEPortal|||Yes||
|155|CMImprovementActionChange||||
|156| FilteringUrlClick||||
|157| MipLabelAnalyticsAuditRecord||||
|158| FilteringEntityEvent||||
|159| FilteringRuleHits||||
|160| FilteringMailSubmission||||
|161| LabelExplorer||||
|162| MicrosoftManagedServicePlatform||||
|163| PowerPlatformServiceActivity|||Yes||
|164| ScorePlatformGenericAuditRecord||||
|165| TimeTravelFilteringDocMetadata||||
|166| Alert||||
|167| AlertStatus||||
|168| AlertIncident||||
|169| IncidentStatus||||
|170| Case||||
|171| CaseInvestigation||||
# 監査ログの保持ポリシーについて
Advanced Audit が含まれているライセンスがあれば、最長 1 年間のログの保持ができ、さらに追加のアドオン ライセンスがあれば、最長 10 年間のログ保存が設定できます。ただし、[Docs](
https://docs.microsoft.com/ja-jp/microsoft-365/compliance/audit-log-retention-policies?view=o365-worldwide#create-and-manage-audit-log-retention-policies-in-powershell) に記載がある通り、UI から保持ポリシーを設定できるログの種類は限定されているためその他のログも含めて、保持設定をする場合には、別途 PowerShell での設定が必要です。

## PowerShell での保持ポリシー設定例
以下の PowerShell では、1 番から 147 番までの範囲で、Docs に記載があるか、UI 上の保持ポリシーで対象として設定できるログの種類を対象に、ユーザー指定なしで、保持ポリシーを設定するものとなっています。
```
Connect-IPPSSession -UserPrincipalName xxxx@xxxx.onmicrosoft.com

$range=@();
$i=1;
$skip=@(5,26,27,74,79,80,104,108);
while ($i -le 109){if(!$skip.Contains($i)){$range+=$i;}$i++;}
$range+=113,147
New-UnifiedAuditLogRetentionPolicy -name "1Year Policy for All" -RetentionDuration TwelveMonths -priority 200 -recordtypes $range
```

# 監査ログの保持ポリシーが有効となるタイミングについて
ログが生成されたタイミングでのライセンス有無および保持ポリシーに従ってログが保持されるようです。なので現在残っているログに対して、今、保持ポリシーを適用しても期間を延長できない一方、ライセンスが切れた後も、当時ライセンスが有効だった期間のログについては、当時のポリシーに従ってログが保持されます。保持する期間が既定の 90 日、6 カ月、9 カ月、1 年、10 年の 5 種類のパターンしかないことから、ログ生成時に保持ポリシーに応じて、ログの入れ物を論理的に分けていて、入れ物ごとの有効期間で、古いログの切り捨てを行っているものと想像できます。

# Compliance Center 上で保持ポリシーの操作できなくなった場合
PowerShell で公開されていないドラフト段階の RecordType に対する保持ポリシーを設定していて、その RecordType の内部の名称が変更となった場合、Compliance Center の[監査保持ポリシー](https://compliance.microsoft.com/auditlogsearch?viewid=Retention)の画面で、保持ポリシーの一覧や新規監査保持ポリシーの作成ができなくことがあります。そういった場合、PowerShell から古い RecordType の名称を指定している監査ログの保持ポリシーを削除する必要があります。
```
Connect-IPPSSession -UserPrincipalName xxxx@xxxx.onmicrosoft.com
#削除コマンドレットで pending deletion state になる
Remove-UnifiedAuditLogRetentionPolicy -Identity "1Year Policy for All"
#再度 -ForceDeletion をつけて削除コマンドレットを実行することで、即時削除できる
Remove-UnifiedAuditLogRetentionPolicy -Identity "1Year Policy for All" -ForceDeletion
```
PowerShell 上 Get-UnifiedAuditLog の操作もエラーとなるので、ポリシーの削除に当たっては、ポリシーの名称を把握している必要があります。もし過去作成した監査ログのポリシー名称が分からなくなった場合には、[こちら](https://github.com/YoshihiroIchinose/E5Comp/blob/main/Office365Audit.md)のページを参考に、SecurityComplianceCenterEOPCmdlet の監査ログを抽出し、Operation が New-UnifiedAuditLogRetentionPolicy となっているログの Parameters から -Name の引数を参照します。
