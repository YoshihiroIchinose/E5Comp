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
 
#Fields Adjustment
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
