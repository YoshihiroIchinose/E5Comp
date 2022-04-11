# Office 365 Management API を利用した監査ログの取得について

````
$AppClientID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx"
$ClientSecretValue = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
$TenantGUID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx"
$tenantdomain = "xxxxxx.onmicrosoft.com"
$OutputPath = "C:\Users\user01\Desktop\MGMTLog\"

$APIResource ="https://manage.office.com"
$BaseURI = "$APIResource/api/v1.0/$tenantGUID/activity/feed/subscriptions"
$Subscription = "Audit.General"
#$Subscription = "Audit.AzureActiveDirectory"
#$Subscription = "Audit.Exchange"
#$Subscription = "Audit.SharePoint"
#$Subscription = "DLP.All"

$today=(Get-Date)
$daysdiff=3 #1-7
$end=$today.AddDays(-$daysdiff+1).Date.ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ss")
$start = $today.AddDays(-$daysdiff).Date.ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ss")
$filter="startTime="+$start+"&endTime="+$end
$filename = ($OutputPath + $Subscription + "_" + $today.AddDays(-$daysdiff).Date.ToString("yyyy-MM-dd") + ".csv")

# アクセス トークンの取得
$body = @{grant_type="client_credentials";resource=$APIResource;client_id=$AppClientID;client_secret=$ClientSecretValue}
try{
    $oauth = Invoke-RestMethod -Method Post -Uri "https://login.microsoftonline.com/$tenantdomain/oauth2/token?api-version=1.0" -Body $body -ErrorAction Stop
    $OfficeToken = @{'Authorization'="$($oauth.token_type) $($oauth.access_token)"}
} catch {
    write-host -ForegroundColor Red "Invoke-RestMethod failed." + $error[0]
    exit
}

#サブスクリプションの確認
$subs = Invoke-WebRequest -Headers $OfficeToken -Uri "$BaseURI/list" -UseBasicParsing |ConvertFrom-Json
$enabled=$false
foreach($sub in $subs){
	if($sub.ContentType.ToLower() -eq $Subscription.ToLower() -and $sub.Status -eq "enabled"){$enabled=$true}
}

#サブスクリプションが有効でなければ作成
if(!$enabled){
$response = Invoke-WebRequest -Method Post -Headers $OfficeToken -Uri "$BaseURI/start?contentType=$Subscription" -UseBasicParsing -ErrorAction Stop
}

#ログの集約のURLを取得
$output=""
$NextContentPageURI="$BaseURI/content?contentType=$Subscription&PublisherIdentifier=$TenantGUID&$filter"
while ($NextContentPageURI){
	$response = Invoke-WebRequest -Headers $OfficeToken -Uri $NextContentPageURI -UseBasicParsing
	$output += $response.Content
	$NextContentPageURI = $response.Headers.NextPageUri
}
$uris = (ConvertFrom-Json $output).contentUri

#ログの集約からログの実体を取得
$Logdata=@()
foreach($uri in $uris){
	$Logdata+= Invoke-RestMethod -Uri $uri -Headers $Officetoken -Method Get
}

#取得したデータを CSV で出力するために Operation の種類ごとに最初の 1 つ目のアイテムから含まれている列を取得し集計
$OperationTypes=$Logdata|Group-Object Operation
$FieldName=@()
foreach($Operation in $OperationTypes){
    $FieldsinLog=$Operation.Group[0]|get-member -type NoteProperty
    foreach($f in $FieldsinLog){
      if($FieldName.Contains($f.Name) -eq $false) { $FieldName+=$f.Name}
    }
}

#取得した列ごとに CSV で出力するために、Script Block を用意、値が省略されないように、ConvertTo-Json を挟む
$Fields=@()
foreach($f in $FieldName){
$sb=[scriptblock]::Create('$att=$d.'+$f+';if($att.GetType().Name -eq "Object[]" -or $att.GetType().Name -eq "PSCustomObject"){ConvertTo-Json -Compress -Depth 10 $att} else {$att}')        
$Fields+=@{Name=$f;Expression=$sb}
}

#取得したデータを CSV で出力するために、1 アイテムごとに全列の値を出力
$csv=@();
foreach($d in $Logdata){
    $output=$d|Select-Object -Property $Fields
    $csv+=$output
 }

#取得したデータを CSV で出力する
$csv|Export-Csv -Path $filename -NoTypeInformation -Encoding UTF8
````
