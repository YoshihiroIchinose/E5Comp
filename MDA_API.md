# Defender for Cloud Apps のファイル ページの情報を API 経由で取得する
本サンプル スクリプトは、Microsoft 365 Defender の[ファイル ページ](https://security.microsoft.com/cloudapps/files)の情報を、PowerShell を使った API アクセスで取得するものです。[ファイル ページ](https://security.microsoft.com/cloudapps/files)においても、さまざまなフィルタ条件を設定し、SPO/OD4B や接続したクラウド ストレージの中から条件に合致したファイル情報を一覧で見ることができます。また Export の機能で UI 上からも CSV 形式でデータを取得することが可能ですが、最大 5,000 件までの制限があります。API を通じたアクセスを行った場合、最大 100 件のデータ取得を繰り返すことで、条件に合致したファイルの情報を 5,000 件を超えて取得することも可能です。
## 事前準備
### API トークン
Defender for Cloud Apps の[API トークン](https://security.microsoft.com/cloudapps/settings?tabid=apiTokens)のページにアクセスし、64 文字の英数字で構成されるトークンを事前に取得しておきます。取得後は再表示されず、以前のトークンが分からなくなった場合には、トークンの再発行が必要となる点に注意します。
### 検索条件を示したフィルタ設定の取得
ファイルを取得する上では、絞り込みに利用するフィルタ条件を設定する必要があります。Microsoft 365 Defender の[ファイル ページ](https://security.microsoft.com/cloudapps/files)で、フィルタを行うと、URL に、フィルタ条件が組み込まれるため条件によって、どういった属性が、どういった値でフィルタされるか確認しながら、スクリプトに記載の例も参考にしながら、JSON 形式で定義します。Microsoft 365 Defender の[ファイル ページ](https://security.microsoft.com/cloudapps/files)でブラウザの開発者ツールを用いて、https://security.microsoft.com/apiproxy/mcas/cas/api/v1/files/count/ への POST 要求の中身を見ることで、JSON 形式のフィルタ文字列を直接確認することもできます。スクリプトの例にある service が 20892 であるという条件は、SharePoint Online 上のファイルに絞る条件となっています。その他、汎用的な Defender for Cloud Apps に対する PowerShell のコマンドレットおよび、サンプル コードは、[こちらのページ](https://github.com/microsoft/MCAS)で確認できます。
## スクリプト本体
````
#Parameters
#should be replaced by the tenant domain
$Uri="https://xxxxx.portal.cloudappsecurity.com/api/v1/files/"

#64 chacters, should be obtained via MDA API Token page
$Token="xxxxxxxxx"

#Filter for finding protected files in SPO withoug sensitivity labels
$filter='{"fileScanLabels":{"eq":["^Azure RMS encrypted file"]},"service":{"eq":[20892]}}'

#Filter for finding files in SPO shared with some specific domain users
#$filter='{"service":{"eq":[20892]},"collaborators.withDomain":{"eq":["hotmail.com","gmail.com","icloud.com"]}}'

#Filter for finding files in SPO with some specific sensitivity labels
#$filter='{"service":{"eq":[20892]},"fileLabels":{"eq":["個別保護","社外秘"]}}'

#Filter for finding files shared externally
#$filter='{"sharing":{"eq":["i:4","i:3","i:2"]}'

#Amount of files to retrieve
$ResultSetSize=300

#ouput csv file
$OutputFile=[System.Environment]::GetFolderPath("Desktop")+"\MDAFiles.csv"

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
	  "sortField"="modifiedDate"
	  "sortDirection"="desc"
  }
  $res=Invoke-RestMethod -Uri $Uri -Method "Post" -Headers $headers -Body $Body
  $output+=$res.data
}
 
#Fields Adjustments as script blocks to remove actions, convert createDate/modifiedData into LocalDateTime, and expand recursive Json text
$FieldName= $output[0] |get-member -MemberType NoteProperty|foreach-object{$_.Name.ToString()}
$FieldName={$FieldName}.Invoke()
$FieldName.Remove("actions")
$Fields=@()
foreach($f in $FieldName){
     if($f -eq "createdDate" -or $f -eq "modifiedDate"){
	    $sb=[scriptblock]::Create('$att=$_.'+$f+';$att=([Int64]$att)/1000;([datetimeoffset]::FromUnixTimeSeconds($att)).LocalDateTime')
	    $Fields+=@{Name=$f.ToString();Expression=$sb}
	    continue
	}
     $sb=[scriptblock]::Create('$att=$_.'+$f+';if($att.GetType().Name -eq "Object[]" -or $att.GetType().Name -eq "PSCustomObject"){ConvertTo-Json -Compress -Depth 10 $att} else {$att}')
    $Fields+=@{Name=$f.ToString();Expression=$sb}
  }

#Save as a csv file
$csv=@()
foreach($row in $output){
	$csv+=$row|Select-Object -Property $Fields
}
$csv|Export-Csv -Path $OutputFile -NoTypeInformation -Encoding UTF8
````
