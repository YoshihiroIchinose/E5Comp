## 準備
1. Azure 環境にて Azure Automation アカウントを作成
2. Azure Automation アカウントにて、"モジュール" -> "ギャラリーを参照"から、以下のモジュールを追加する。   
(ランタイム バージョンは 5.1)   
  SharePointOnline.CSOM    
3. Azure Automation アカウントの"資格情報"->"資格情報の追加"で、指定の SharePoint Online サイトに投稿権限があるアカウントの ID とパスワードを "Office 365" という名称で登録しておく。
4. Azure Automation アカウントの"Runbook"->"Runbook の作成"で PowerShell、ランタイム バージョンの 5.1 の Runbook を作成する
5. 作成した Runbook に以下のスクリプトをコピー & ペーストする
6. 適宜スクリプト内の SharePoint Site の URL および、ファイル保存先の相対 URL (FQDN を除いたもの) を書き変え、保存し、公開する
7. 作成した Runbook を"開始"し、動作を確認する

````
#MDA URL
$Base="https://xxxx.portal.cloudappsecurity.com/"
#Target CSV location in SPO
$siteUrl="https://xxxx.sharepoint.com/sites/CustomNotification/"
$targeturl ="/sites/CustomNotification/Shared Documents/ExternalUsers.csv"
#Internal domain list
$InternalDomains=@("xxxx.onmicrosoft.com")
#Maximum files to retrieve
$ResultSetSize=3000

#fixed parameters (No need to modify)
$Credential = Get-AutomationPSCredential -Name "Office 365"
$Token=Get-AutomationVariable -Name "MDAToken"
#a filter condition for getting externally shared files from SPO and OD4B
$filter='{"service":{"eq":[20892,15600]},"sharing":{"eq":[2,3,4]}}'
#Temporal output location
$OutputFile=[System.Environment]::GetFolderPath("Desktop")+"\ExternalUsers.csv"
#Maximum files to retrieve at a sigle query
$batchSize=100

Function AddMember{
    Param($a,$b,$c)
    add-member -InputObject $a -NotePropertyName $b  -NotePropertyValue $c
}

Function IsExternalDomains{
    Param($a)
    $a=$a.ToLower()
    foreach($i in $InternalDomains){
        if($a.EndsWith($i)){return $false}
    }
    return $true
}

$loopcount = [int][Math]::Ceiling($ResultSetSize / $batchSize)
$headers=@{"Authorization" = "Token "+$Token}
$output=@()
"Start getting externally shared files from MDA"
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
    $Uri=($base+"/api/v1/files/").Replace("//api/","/api/")
do {
        $retryCall = $false
	    "Loop: $i, From " +$i*$batchSize
    try {
		    $res=Invoke-RestMethod -Uri $Uri -Method "Post" -Headers $headers -Body $Body
    }
    catch {
            if ($_ -like '504' -or $_ -like '502' -or $_ -like '429') {
	        $retryCall = $true
            Start-Sleep -Seconds 5
            }
            ElseIf ($_ -match 'throttled') {
                $retryCall = $true
                Start-Sleep -Seconds 60
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

$userlist=@()
$GroupsWithExternalUsers=@{}
foreach($row in $output){
    $groupIds=@()
	foreach($c in $row.collaborators){
		If($c.type -eq 2 -and $c.accessLevel -eq 2){#Group with external users
			 $group = New-Object -TypeName PSObject
			 AddMember $group "Item" $c.name
			 AddMember $group "Id" $c.id
			$groupIds+=$group
		}
		If($c.type -eq 1 -and $c.accessLevel -eq 2){#direct assignments of external users 
			If($c.name -eq "NT Service\SPTimerV4"){continue}
			 $line = New-Object -TypeName PSObject
			 AddMember $line "Site" $row.siteCollection
		     AddMember $line "Item" $row.filePath
			 AddMember $line "User" $c.name
			 AddMember $line "Email" $c.email
			 $userList+=$line
		}
	}
	If($groupIds.count -gt 0){
		$GroupsWithExternalUsers[$row.sitePath]=$groupIds
	}
}
"Files with direct assignments of external users:"+$userList.count
"Sites with SPO Groups including external users:"+$GroupsWithExternalUsers.count

foreach($Site in $GroupsWithExternalUsers.Keys){
	foreach($g in $GroupsWithExternalUsers[$Site]){
		$groupId="$Site|$($g.Id)"
		$groupId= [System.Web.HttpUtility]::UrlEncode($groupId) 
		$appId=20892
		If($Site.StartsWith("/personal/")){$appId=15600}
        "Getting external users from $($g.Item) in $Site"
		$Uri=($base+"/api/v1/get_group/?appId=$appId&groupId=$groupId&limit=100").Replace("//api/","/api/")
		do {
		        $retryCall = $false
		    try {
			        $res=Invoke-RestMethod -Uri $Uri -Method "GET" -Headers $headers
		    }
		    catch{
		            if ($_ -like '504' -or $_ -like '502' -or $_ -like '429') {
			            $retryCall = $true
		                Start-Sleep -Seconds 5
                    }
		            ElseIf ($_ -match 'throttled') {
		                $retryCall = $true
		                Start-Sleep -Seconds 60
		            }
		            else {
		                throw $_
		            }
		        }
		}
		while ($retryCall)
		Foreach($u in $res.group.membersList){
		    $line = New-Object -TypeName PSObject
		    if($u.emailAddress -ne $null -and (IsExternalDomains($u.emailAddress))){
			    AddMember $line "Site" $Site
			    AddMember $line "Item" $g.Item
			    AddMember $line "User" $u.Name
			    AddMember $line "Email" $u.emailAddress
			    $userList+=$line
		    }
		}
	}
}
"Total external users found:"+$userList.count
$userList|Export-Csv -Path $OutputFile -NoTypeInformation -Encoding UTF8

#Uploading a csv file to SPO
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
            Start-Sleep -s 5
    }
}
$ctx.Credentials=$cre

$fs = new-object System.IO.FileStream($OutputFile ,[System.IO.FileMode]::Open,[System.IO.FileAccess]::Read)
[Microsoft.SharePoint.Client.File]::SaveBinaryDirect($ctx,$targeturl , $fs, $true)
$fs.Close()
$ctx.Dispose()
"Upload completed."
````
