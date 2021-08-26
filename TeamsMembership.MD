# 概要
本スクリプトは、ネストは考慮せず特定のグループに属する監視対象のユーザーが、許可されていない Teams への参加があれば、それらを CSV 出力するスクリプトになります。Connect に利用する ID は、チームのメンバー情報を参照できる必要があります。[参考:グループ管理者ロール](https://docs.microsoft.com/ja-jp/azure/active-directory/roles/permissions-reference#groups-administrator)

## 事前準備
管理者権限の PowerShell で一度以下を実行
```
Install-Module -Name AzureAD
Install-Module -Name MicrosoftTeams
```

## 監視対象のユーザーが許可されていない Teams のチームに参加していないかチェックするスクリプト サンプル
```
#監査対象のグループ(ネストは考慮しない)
$GroupforAudit ="sg-Legal"
#監査対象のグループのメンバーが参加許可されているTeams チーム
$AllowedTeams=("Contoso Team", "Ask HR", "Operations")
#接続に利用するID / パスワード
$Id="xxx@xxx.onmicrosoft.com"
$Password = "xxxxx"
$OutputFolder=[System.Environment]::GetFolderPath("Desktop")+"\"

#Credentialの生成
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Connect-AzureAD -credential $Credential

#監視対象のグループを取得
$g=Get-AzureADGroup -SearchString $GroupforAudit

#許可されているO365 Groupの詳細を取得
$AllowedTeamsGuid=@();
Foreach($t in $AllowedTeams){
	$r=Get-AzureADGroup -SearchString $t
	$AllowedTeamsGuid+=$r.ObjectId
}

#許可されたものを除く Teams のチームを全件取得
Connect-MicrosoftTeams -credential $Credential
$Teams=Get-Team -NumberOfThreads 20
$TeamsGuid=@()
Foreach($tg in $Teams){
	If(!$AllowedTeamsGuid.IndexOf($tg.GroupId) -ne -1)
	{$TeamsGuid+=$tg.groupId.ToString()}
}

#対象のユーザーの取得
$UsersforAudit=Get-AzureADGroupMember -ObjectId $g.ObjectId -All $true|?{$_.UserType -eq "Member"} 

$csv=@()
Foreach($u in $UsersforAudit){
	#監査対象のユーザーのTeamsのメンバーシップを取得
	$membership=Get-AzureADUserMembership -ObjectId $u.ObjectID -All $true|?{$_.ObjectType -eq "Group"}|?{$TeamsGuid.IndexOf($_.ObjectId) -ne -1}

        #許可されていないグループへの参加があれば記録
	Foreach($m in $membership){
		$line = New-Object PSObject | Select-Object UserName, UPN, TeamName, TeamGuid
		$line.UserName=$u.DisplayName
		$line.UPN=$u.UserPrincipalName
		$line.TeamName=$m.DisplayName
		$line.TeamGuid=$m.ObjectID
		$csv+=$line
		}
}

#CSV出力
$csv|Export-csv -Path ($OutputFolder+"UnAllowedMembership"+(Get-Date -Format "yyyyMMdd-HHmm")+".csv") -Encoding UTF8 -NoTypeInformation

```

## 出力サンプル
```
"UserName","UPN","TeamName","TeamGuid"
"Joni Sherman","JoniS@xxxx.OnMicrosoft.com","Sales and Marketing","8cd8e24a-71da-4891-b5ce-b38b7905944d"
"Joni Sherman","JoniS@xxxx.OnMicrosoft.com","Mark 8 Project Team","ac6d81e5-0f73-4127-9bde-228c4b9bc20f"
"Joni Sherman","JoniS@xxxx.OnMicrosoft.com","Retail","cc1265cc-842d-4dae-89d3-08caf2f5da77"
"Irvin Sayers","IrvinS@xxxx.OnMicrosoft.com","Digital Initiative Public Relations","e88fe2fc-9a3e-48f0-8a37-78a3db263866"
"Irvin Sayers","IrvinS@xxxx.OnMicrosoft.com","U.S. Sales","04a1f966-ff6f-4351-8418-23a28700528d"
"Grady Archie","GradyA@xxxx.OnMicrosoft.com","Contoso","09645b11-46ce-49bf-ad5d-53a9ac8feba5"
````
