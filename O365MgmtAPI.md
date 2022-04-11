# Office 365 Management API を利用した監査ログの取得について
Search-UnifiedAuditLog よりも大規模に対応する Office 365 Management API を利用した PowerShell からの Office 365 監査ログの取得サンプルです。事前準備として、[こちら](https://docs.microsoft.com/ja-jp/office/office-365-management-api/get-started-with-office-365-management-apis)の設定に従い、API を利用するための、アプリの登録、権限の設定、Client Secret の取得を事前に行っておく必要があります。Office 365 Management API を利用してログを取得する場合、細かな抽出条件を指定することはできず、Office 365 の監査ログがカテゴリ分けされた、5 種類のログの中から一つを選択して、特定の範囲時間範囲内のログを取得することになります。また仕組みとしては、これら 5 種類のログ毎に Subscription を開始すると、5 種類のログ毎に、ある程度の数のログがまとまった BLOB ファイルが溜まって行きます。最大 7 日前までに遡って、一回のクエリで最大 24 時間の範囲のログの BLOB の URL が取得でき、その URL からログの中身が取得できます。ログの取得そのものは、以下の 1-5 の手順で完了し、CSV 出力するために、6-9 の手順でログ全体の中から 1 階層目の列を全種類取得し、ログ アイテムごとに全列の値を出力するようにしています。10 手順はおまけでテーブル変換した .xlsx に変換する手順を入れています。

1. Application ID と Client Secret を使ってアクセス トークンを取得する (x1 Post 要求)
2. 取得したログの Subscription が有効か確認する (x1 Get 要求)
3. 有効でなければ Suscription を開始する (x1 Post 要求)
4. 対象となる日付の期間を指定し、複数のログの BLOB ファイルの URL を取得する (x1 Get 要求だが、対象が多いとページ分割され複数回の可能性あり)
5. 取得した BLOB ファイルの URL に一つずつアクセスし中身を取得して連結する (対象の BLOB ファイルごとに GET 要求)
6. CSV で出力するために Operation の種類ごとに最初の 1 つ目のアイテムから含まれている列の種類をすべて把握する
7. 取得したログを CSV で出力する際、値が省略されないように、ConvertTo-Json の処理を入れた Script Block を用意する
8. 取得したデータを CSV で出力するために、1 アイテムごとにログ全体の全列の値を出力
9. 取得したデータを CSV で出力する
10. CSV 出力したファイルを、テーブル変換した .xlsx として保存しなおす

なお 1 分間に、2,000 を超える要求がある場合には、規模によってリクエストがエラーになることがあります。詳細は[こちら](https://docs.microsoft.com/ja-jp/office/office-365-management-api/office-365-management-activity-api-reference#api-throttling)。

# PowerShell のサンプル コード
````
$AppClientID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx"
$ClientSecretValue = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
$TenantGUID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx"
$tenantdomain = "xxxxxx.onmicrosoft.com"
$OutputPath = "C:\Users\user01\Desktop\MGMTLog\"

$APIResource ="https://manage.office.com"
$BaseURI = "$APIResource/api/v1.0/$tenantGUID/activity/feed/subscriptions"

#以下の 5 種類の中からいずれか一つを選択
$Subscription = "Audit.General"
#$Subscription = "Audit.AzureActiveDirectory"
#$Subscription = "Audit.Exchange"
#$Subscription = "Audit.SharePoint"
#$Subscription = "DLP.All"


$today=(Get-Date)
#ログは、7日前の中で、最大 24 時間の範囲で取得できる
#ローカルの日時で1-6 日前のデータを1日分取得する
$daysdiff=3
$end=$today.AddDays(-$daysdiff+1).Date.ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ss")
$start = $today.AddDays(-$daysdiff).Date.ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ss")
$filter="startTime="+$start+"&endTime="+$end
$filename = ($OutputPath + $Subscription + "_" + $today.AddDays(-$daysdiff).Date.ToString("yyyy-MM-dd") + ".csv")

# 1.アクセス トークンの取得
$body = @{grant_type="client_credentials";resource=$APIResource;client_id=$AppClientID;client_secret=$ClientSecretValue}
try{
	$oauth = Invoke-RestMethod -Method Post -Uri "https://login.microsoftonline.com/$tenantdomain/oauth2/token?api-version=1.0" -Body $body -ErrorAction Stop
	$OfficeToken = @{'Authorization'="$($oauth.token_type) $($oauth.access_token)"}
}
catch {
    	write-host -ForegroundColor Red "Invoke-RestMethod failed." + $error[0]
    exit
}

# 2.サブスクリプションの確認
$subs = Invoke-WebRequest -Headers $OfficeToken -Uri "$BaseURI/list" -UseBasicParsing |ConvertFrom-Json
$enabled=$false
foreach($sub in $subs){
	if($sub.ContentType.ToLower() -eq $Subscription.ToLower() -and $sub.Status -eq "enabled"){$enabled=$true}
}

# 3.サブスクリプションが有効でなければ作成
if(!$enabled){
	Invoke-WebRequest -Method Post -Headers $OfficeToken -Uri "$BaseURI/start?contentType=$Subscription" -UseBasicParsing -ErrorAction Stop
}

#4.対象となる日付の期間を指定し、複数のログの BLOB ファイルの URL を取得する
$output=""
$next="$BaseURI/content?contentType=$Subscription&PublisherIdentifier=$TenantGUID&$filter"
while ($next){
	$repsonse=Invoke-WebRequest -Headers $OfficeToken -Uri $next -UseBasicParsing
	$output += $repsonse.Content
	$next = $response.Headers.NextPageUri
}
$uris = (ConvertFrom-Json $output).contentUri

#5.取得した BLOB ファイルの URL に一つずつアクセスし中身を取得して連結する
$Logdata=@()
foreach($uri in $uris){
	$Logdata+= Invoke-RestMethod -Uri $uri -Headers $Officetoken
}

#6.CSV で出力するために Operation の種類ごとに最初の 1 つ目のアイテムから含まれている列の種類をすべて把握する
$OperationTypes=$Logdata|Group-Object Operation
$FieldName=@()
foreach($Operation in $OperationTypes){
    $FieldsinLog=$Operation.Group[0]|get-member -type NoteProperty
    foreach($f in $FieldsinLog){
      if(!$FieldName.Contains($f.Name)) { $FieldName+=$f.Name}
    }
}

#7.取得したログを CSV で出力する際、値が省略されないように、ConvertTo-Json の処理を入れた Script Block を用意する
$Fields=@()
foreach($f in $FieldName){
$sb=[scriptblock]::Create('$att=$d.'+$f+';if($att.GetType().Name -eq "Object[]" -or $att.GetType().Name -eq "PSCustomObject"){ConvertTo-Json -Compress -Depth 10 $att} else {$att}')        
$Fields+=@{Name=$f;Expression=$sb}
}

#取得したデータを CSV で出力するために、1 アイテムごとにログ全体の全列の値を出力
$csv=@();
foreach($d in $Logdata){
    $output=$d|Select-Object -Property $Fields
    $csv+=$output
 }

#9.取得したデータを CSV で出力する
$csv|Export-Csv -Path $filename -NoTypeInformation -Encoding UTF8

#10.CSV 出力したファイルを、テーブル変換した .xlsx として保存しなおす
$excel = new-Object -com excel.application
$excel.visible = $false
$book = $excel.Workbooks.open($filename)
$book.ActiveSheet.ListObjects.Add(1,$book.ActiveSheet.Range("A1").CurrentRegion ,$null,1).Name = "テーブル1"
$book.SaveAs($filename.Replace(".csv",".xlsx"),51)
$book.close()
$excel.quit()
````
