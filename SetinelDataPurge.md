# Microsoft Sentinel (Log Analytics) の特定のログを削除する
現在プレビュー機能で、秘密度ラベルの操作に関するログを Microsoft Sentinel に取り込めるデータ コネクタがプレビューで提供開始されています。     
[Microsoft Purview Information Protection から Microsoft Sentinel にデータをストリーミングする](https://learn.microsoft.com/ja-jp/azure/sentinel/connect-microsoft-purview)

このデータ コネクタにより、数クリックで秘密度ラベルに関する操作ログが Microsoft Sentinel に取り込めるようになり、
秘密度ラベルに関する操作を Kusto クエリを介して Power BI のデータ ソースとして分析するダッシュボードを作成することや、
Logic Apps を通じて、ラベルを削除したり、ラベルを下げるなどの操作をした場合、上司や関係者に通知するなど
カスタムの自動対応の実装が行いやすくなります。一方で、AIP UL Client を利用していると、 AIPHeartBeat のログも取り込まれるため
この AIPHeartBeat のログのみ削除したいという要望もあるかと思います。そのため、このページでは、Azure Automation を利用して、
Log Anlytics からこの AIPHeartBeat のログのみを削除するサンプル コードを共有します。

## 事前準備
1. システム割り当てマネージド ID の Azure Automation アカウントを作成する
2. Log Analytics で上記で作成した、システム割り当てマネージド ID に "Data Purger" の役割を割り当てる
3. Azure Automation アカウントで、PowerShell 5.1 を利用する新しい Runbook を作成し、以下のサンプル コードを貼り付ける
4. サブスクリプション ID、リソース グループ名、Log Analytics のワークスペース名を実環境に書き換える
5. Runbook を保存、発行し、適宜スケジュール実行する

## 注意事項
1. [REST API リファレンス](https://learn.microsoft.com/ja-jp/rest/api/loganalytics/workspace-purge/purge?tabs=HTTP)によると、
データの消去は、制限なく認められるのものではなく、1 時間当たり 50 要求に制限されること、GDPR への準拠に必要な消去操作のみがサポートされ、
GDPR 準拠に関連しない要求は拒否される可能性があることが示されています。
2. 消去の操作をキックした後、即座に消去が行われるわけではなく、ある程度時間をかけて実行されます。
以下のような REST AIP を通じて、スタータスを確認することは可能です。    
[Get Purge Status](https://learn.microsoft.com/ja-jp/rest/api/loganalytics/workspace-purge/get-purge-status?tabs=HTTP)


## Azure Automation でのサンプル コード
````
$subscription ="xxxxxx"
$resourcegroup="xxxxx"
$workspace="xxxxx"
$resource= "?resource=https://management.azure.com/" 
$url = $env:IDENTITY_ENDPOINT + $resource
$headers = @{
    "X-IDENTITY-HEADER" = $env:IDENTITY_HEADER
    "Metadata"="True"
}
$accessToken = Invoke-RestMethod -Uri $url -Method 'GET' -Headers $headers

$uri="https://management.azure.com/subscriptions/$subscription/resourceGroups/$resourcegroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/purge?api-version=2020-08-01"
$headers = @{
    "Content-Type" = "application/json"
    "Authorization"=("Bearer {0}" -f $accessToken.access_token)
}
$body = @"
{
  "table": "MicrosoftPurviewInformationProtection",
  "filters":[{"column":"RecordType","operator":"==","value":97}]
}
"@
Invoke-WebRequest -Method "Post" -Uri $uri -Headers $headers -Body $body -UseBasicParsing
````
