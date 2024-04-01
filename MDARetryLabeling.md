## 今後利用できなくなる可能性あり
本スクリプトは、従来型の MDA の全機能が利用可能であったレガシーのトークンを利用したものです。今後推奨の OAuth 2.0 のトークンでは、権限付与の範囲が厳格化され、本スクリプトで利用している API は動作しなくなります。そのため本スクリプトも、今後動作させることができなくなる可能性があります。参考：[MDA トークン](https://learn.microsoft.com/ja-jp/defender-cloud-apps/api-authentication)

## スロットリング時の動作について
MDA では、1 分間に 30 Call という制限がありますが、MDA の環境によって、この制限を超えた API 呼び出しの際、即時スロットリングされたというエラーが返ってくる非同期的な動作の場合と、1 分間応答が待たされて、正しく結果が返ってくるという同期的な動作の 2 種類の動作があります。本スクリプトは、同期的な動作の環境でテストしているため、非同期的な動作をする環境でのエラー ハンドリングが正しく検証できておりません。

# Defender for Cloud Apps でのラベル適用のガバナンス アクションをリトライする
本サンプル スクリプトは、Microsoft 365 Defender の[ガバナンス ログ](https://security.microsoft.com/cloudapps/governance-log)の情報を、
PowerShell を使った API アクセスで取得し、失敗したままになっている秘密度ラベル適用をリトライするものです。
具体的には、ガバナンス ログの中から成功・失敗含めてラベル付けのアクションを直近 8 時間の範囲で、最大 300 件取得し、そのうち以下のものを除いてリトライを実施します。適宜バッチの実行頻度に応じて、ガバナンス ログの取得件数や、対象外とするログの範囲を調整下さい。   
- 新しいログで既にラベル付けに成功しているファイルに関する操作
- 既にラベルが付与されていることにより失敗しているもの
- 1 時間以内に失敗しているもの
- 24 時間以上前に作成されたファイル
- 同じファイルに対する重複したリトライ (2023/12/25 更新)

バッチ実行に当たっては、本スクリプトを、Azure Automation 上に配置するか、インターネットに接続可能で、PowerShell が動作する常時稼働の Windows マシンで、スケジュール実行します。(特に特殊なモジュールのインストールなどは不要。)    
---(2023/12/25 更新)---   
なお、Box のダウンロード無効化のロック時や、各種障害等によりラベル付けが失敗・内部エラーとなる場合、ガバナンス ログでは、ShouldRetry = false となっているがこれらもリトライするように変更(ShouldRetryの判定条件を # でコメント アウト)。また同じファイルで複数のラベル付け失敗のログがある場合に、1 回のスクリプト実行で、重複してラベル付けをリトライをしないように修正。
## Defender for Cloud Apps のラベル付け動作
Defender for Cloud Apps のラベル付けはでは以下の動作が行われています。
1. アクティビティとして新しいファイルの作成・アップロードがログに確認される
1. ファイル ページで新しいファイルが認識される
1. ファイル ポリシーに合致していた場合、同ファイルに対するガバナンス ログが記録され、ガバナンス アクションとしてラベル付けが試行される
1. ファイルが編集中の場合など初回の試行が失敗した場合、ガバナンス アクションは保留状態となる
1. 初回の施行後、15 分間隔で 3 回リトライし、おおよそ 45 分間で 4 回の試行がすべて失敗した場合、ガバナンス アクションは失敗状態となる   
   (ファイルがずっと編集中であった場合、"保護されたファイルをアップロードできませんでした"というエラーで失敗状態となる)
1. 一度ガバナンス アクションが失敗状態となった場合、以後同ファイルの更新があっても、ラベル付けはリトライされない
1. ガバナス ログからの手動のリトライもしくは本スクリプトのリトライにより、ラベル付けが失敗したままになっているファイルのラベル付けを再度試行することが可能

## 事前準備
### API トークン
Defender for Cloud Apps の [API トークン](https://security.microsoft.com/cloudapps/settings?tabid=apiTokens)のページにアクセスし、
64 文字の英数字で構成されるトークンを事前に取得しておきます。取得後は再表示されず、以前のトークンが分からなくなった場合には、トークンの再発行が必要となる点に注意します。

## スクリプト本体
````
#Parameters
#should be replaced by the tenant domain
$Uri="https://xxxxx.portal.cloudappsecurity.com/api/v1/governance/"

#64 chacters, should be obtained via MDA API Token page
$Token="xxxxxxxxx"

#Filter for finding governance actions related to labeling within last 8 hours
$t=[datetimeoffset]::Now.AddHours(-8).ToUnixTimeMilliseconds()
$filter='{"status":{"eq":[true, false]},"timestamp":{"gte":'+$t+'},"type":{"eq":["RmsProtectTask"]}}'

#Amount of governance actions to retrieve
$ResultSetSize=300

#Getting files 
$batchSize=100 # the fixed limit for a single request
$loopcount = [int][Math]::Ceiling($ResultSetSize / $batchSize)
$headers=@{"Authorization" = "Token "+$Token}
$output=@()
For($i=0;$i -lt $loopcount; $i++){
	$limit=$batchSize
	if($loopcount -1 -eq $i){$limit=$ResultSetSize % $batchSize}
	if($limit -eq 0){$limit=$batchSize}
	$Body=@{
		"skip"=0 + $i*$batchSize
		"limit"=$limit
		"filters"=$filter
		"sortField"="timestamp"
		"sortDirection"="desc"
		}
	do {
		$retryCall = $false
		try {
			$res=Invoke-RestMethod -Uri $Uri -Method "Post" -Headers $headers -Body $Body
			"Loop: $i, From " +$i*$batchSize +", " + $res.data.Count +" items"
			}
		catch {
			if ($_ -like 'The remote server returned an error: (429) TOO MANY REQUESTS.'){
				$retryCall = $true
				Start-Sleep -Seconds 5
			}
			ElseIf ($_ -match 'throttled'){
				$retryCall = $true
				Start-Sleep -Seconds 60
			}
			ElseIf ($_ -like '504' -or $_ -like '502'){
				$retryCall = $true
				Start-Sleep -Seconds 5
				}
			else {
				throw $_
			}
		}
	}
	while ($retryCall)
	$output+=$res.data
	if($res.data.Count -lt $batchsize){break}
}
"Retrieved " +$output.count+" actions"

$completed=@()
$retried=@()
Foreach($d in $output){
        $skipmessage=@()
	#Skip labeled file
	if($completed.IndexOf($d.targetObjectId) -ne -1){
		$skipmessage+="Labeld file"
	}
	#Skip retried file
	if($retried.IndexOf($d.targetObjectId) -ne -1){
		$skipmessage+="Retried file"
	}
	#Skip success
	if($d.status.isSuccess){
		$completed+=$d.targetObjectId
		$skipmessage+="Successful task"
	}
	#Skip which is not supporsed to be retried #removed this condition
	#if($d.status.shouldRetry -eq $false){$skipmessage+="ShouldNotRetry"}

	#Skip files protected by any solution outside of MDA
	if($d.status.statusMessage.Contains("already protected")){
		$completed+=$d.targetObjectId
		$skipmessage+="Already Protected"
	}

	#Skip actions which was taken within last 1 hour
	$initiated=([datetimeoffset]::FromUnixTimeMilliseconds($d.timestamp)).UtcDateTime
	If($initiated -ge (Get-Date).ToUniversalTime().AddHours(-1)){$skipmessage+="Within last 1 hours"}

	#Skip files created over 24 hours before
    	If($d.created.ToDateTime($null) -le (Get-Date).AddHours(-24)){$skipmessage+="Old file"}

	if($skipmessage.count -ge 1){
        	$d.targetObject+","+$d.targetObjectId+",skipped, due to " + ($skipmessage -join ", ")
        	continue
        }

	$d.targetObject+","+$d.targetObjectId+",Retry"
	#Retry should be called with governance log id
	$RetryUri=$Uri+$d._id+"/retry/"
	$res2=Invoke-RestMethod -Uri $RetryUri -Method "Get" -Headers $headers
    	$retried+=$d.targetObjectId
	Start-Sleep -Seconds 1
}

````
