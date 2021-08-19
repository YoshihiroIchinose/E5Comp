#概要
本スクリプトは、特定のグループに属する監視対象のユーザーが、許可されていない Teams への参加があれば、それらを出力するスクリプトになります。

#監視対象のユーザーが許可されていない Teams のチームに参加していないかチェックするスクリプト サンプル

#事前準備
管理者権限の PowerShell で一度以下を実行
```
Install-Module -Name AzureAD
Install-Module -Name MicrosoftTeams
```

#監視対象のユーザーが許可されていない Teams のチームに参加していないかチェックするスクリプト サンプル
```
#監査対象のグループ
$GroupforAudit ="All Employees"
#監査対象のグループのメンバーが参加許可されているTeams チーム
$AllowedTeams=("Contoso Team", "Ask HR", "Operations")
#接続に利用するID
$connectionID="xxxx@xxxx.onmicrosoft.com"

#Teams のチームを全件取得
Connect-MicrosoftTeams -AccountId $connectionID
$Teams=Get-Team -NumberOfThreads 20
$TeamsGuid=@()
Foreach($tg in $Teams){
	$TeamsGuid+=$tg.groupId.ToString()
}

Connect-AzureAD -AccountId $connectionID

#監視対象のグループを取得
$g=Get-AzureADGroup -SearchString $GroupforAudit

#許可されているO365 Groupの詳細を取得
$AllowedTeamsGuid=@();
Foreach($t in $AllowedTeams){
	$r=Get-AzureADGroup -SearchString $t
	$AllowedTeamsGuid+=$r.ObjectId
}

#対象のユーザーの取得
$UsersforAudit=Get-AzureADGroupMember -ObjectId $g.ObjectId -All $true|Where-Object {$_.UserType -eq "Member"} 

#対象のユーザーのメンバーシップを取得
Foreach($u in $UsersforAudit){
	#監査対象のユーザーのTeamsのメンバーシップを取得
	$membership=Get-AzureADUserMembership -ObjectId $u.ObjectID -All $true |Where-Object {$TeamsGuid.IndexOf($_.ObjectId) -ne -1}

         #許可されていないグループへの参加があれば、出力
	Foreach($m in $membership){
		$index=$AllowedTeamsGuid.IndexOf($m.ObjectId)
		If($index -eq -1){$u.DisplayName+"("+$u.UserPrincipalName+") is in "+$m.DisplayName +"("+$m.ObjectID+")"
		}
	}
}
```

#出力サンプル
```
Alex Wilber(AlexW@xxxx.OnMicrosoft.com) is in Sales and Marketing(8cd8e24a-71da-4891-b5ce-b38b7905944d)
Alex Wilber(AlexW@xxxx.OnMicrosoft.com) is in Mark 8 Project Team(ac6d81e5-0f73-4127-9bde-228c4b9bc20f)
Alex Wilber(AlexW@xxxx.OnMicrosoft.com) is in Retail(cc1265cc-842d-4dae-89d3-08caf2f5da77)
Alex Wilber(AlexW@xxxx.OnMicrosoft.com) is in Digital Initiative Public Relations(e88fe2fc-9a3e-48f0-8a37-78a3db263866)
Alex Wilber(AlexW@xxxx.OnMicrosoft.com) is in U.S. Sales(04a1f966-ff6f-4351-8418-23a28700528d)
Alex Wilber(AlexW@xxxx.OnMicrosoft.com) is in Contoso(09645b11-46ce-49bf-ad5d-53a9ac8feba5)
Debra Berger(DebraB@xxxx.OnMicrosoft.com) is in Sales and Marketing(8cd8e24a-71da-4891-b5ce-b38b7905944d)
Debra Berger(DebraB@xxxx.OnMicrosoft.com) is in Mark 8 Project Team(ac6d81e5-0f73-4127-9bde-228c4b9bc20f)
Debra Berger(DebraB@xxxx.OnMicrosoft.com) is in Retail(cc1265cc-842d-4dae-89d3-08caf2f5da77)
Debra Berger(DebraB@xxxx.OnMicrosoft.com) is in Digital Initiative Public Relations(e88fe2fc-9a3e-48f0-8a37-78a3db263866)
Debra Berger(DebraB@xxxx.OnMicrosoft.com) is in Contoso(09645b11-46ce-49bf-ad5d-53a9ac8feba5)
````
