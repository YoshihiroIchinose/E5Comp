````
$Uri="https://xxxx.portal.cloudappsecurity.com/api/v1/files/
$Token="xxxxxxxxxxxxxxxxxxxxxxxx"
$ResultSetSize=10

$loopcount = [int][Math]::Ceiling($ResultSetSize / 100)
$headers=@{"Authorization" = "Token "+$Token}
$output=@()
For($i=0;$i -lt $loopcount; $i++){
  $Body=@{
  "skip"=0 + $i*100
  "limit"=100
  "filters"='{"fileScanLabels":{"eq":["^Azure RMS encrypted file"]},"service":{"eq":[20892]}}'
  "sortField"="modifiedDate"
  "sortDirection"="desc"}
  $res=Invoke-RestMethod -Uri $Uri -Method "Post" -Headers $headers -Body $Body
  $output+=$res.data
}
````
