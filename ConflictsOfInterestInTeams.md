# 概要
本スクリプトは、利益相反するグループ間のコミュニケーションを監視する方法の一つとして、ネストは考慮せず特定のGroupAに属するメンバーとGroup Bに属するメンバーが、許可されていない Teams チームで、メンバーとして共存しているのであれば、それらを列挙するスクリプトです。Connect に利用する ID は、チームのメンバー情報を参照できる必要があります。[参考:グループ管理者ロール](https://docs.microsoft.com/ja-jp/azure/active-directory/roles/permissions-reference#groups-administrator)

## 事前準備
管理者権限の PowerShell で一度以下を実行
```
Install-Module -Name AzureAD
Install-Module -Name MicrosoftTeams
```

## 利益相反するGroupAのメンバーとGroupBのメンバーが共通に参加する Teams チームをチェックするスクリプト サンプル
```
#監査対象のグループ(ネストは考慮しない)
$GroupforAuditA ="sg-Legal"
$GroupforAuditB ="sg-Finance"
#Group AとGroup Bのメンバーの同居が許可されているTeams チーム
$AllowedTeams=("Contoso Team", "Ask HR", "Operations")
#接続に利用するID
$connectionID="xxx@xxx.onmicrosoft.com"

Connect-AzureAD -AccountId $connectionID
#許可されているO365 Groupの詳細を取得
$AllowedTeamsGuid=@();
Foreach($t in $AllowedTeams){
	$r=Get-AzureADGroup -SearchString $t
	$AllowedTeamsGuid+=$r.ObjectId
}

#許可されたものを除く Teams のチームを全件取得
Connect-MicrosoftTeams -AccountId $connectionID
$Teams=Get-Team -NumberOfThreads 20
$TeamsGuid=@()
Foreach($tg in $Teams){
	If(!$AllowedTeamsGuid.IndexOf($tg.GroupId) -ne -1)
	{$TeamsGuid+=$tg.groupId.ToString()}
}

#監視対象のグループを取得
$ga=Get-AzureADGroup -SearchString $GroupforAuditA
$gb=Get-AzureADGroup -SearchString $GroupforAuditB

#Group A のユーザーの取得
$UsersforAuditA=Get-AzureADGroupMember -ObjectId $ga.ObjectId -All $true|Where-Object {$_.UserType -eq "Member"} 

#Group B のユーザーの取得
$UsersforAuditB=Get-AzureADGroupMember -ObjectId $gb.ObjectId -All $true|Where-Object {$_.UserType -eq "Member"} 

#事前にGorup Bのメンバーの全メンバーシップを取得
$UserBTable=@()
$MembershipBTable=@()
Foreach($ub in $UsersforAuditB){
$membershipB=Get-AzureADUserMembership -ObjectId $ub.ObjectID -All $true |Where-Object {$_.ObjectType -eq "Group"} | Where-Object {$TeamsGuid.IndexOf($_.ObjectId) -ne -1}
$UserBTable+=$ub
#2次元配列とするため , を入れる
$MembershipBTable+=,$membershipB
}

#Group A のメンバーと Group B のメンバーが共通に属する Teams チームを見つける
Foreach($ua in $UsersforAuditA){
	#Group A のメンバーのメンバーシップを取得
	$membershipA=Get-AzureADUserMembership -ObjectId $ua.ObjectID -All $true |Where-Object {$_.ObjectType -eq "Group"} | Where-Object {$TeamsGuid.IndexOf($_.ObjectId) -ne -1}
	Foreach($ma in $membershipA){
		Foreach($mb in $MembershipBTable){
			$i=$mb.IndexOf($ma)
			If($i -ne -1){
			$ub=$UserBTable[$MembershipBTable.IndexOf($mb)]
			$ua.DisplayName+"("+$ua.UserPrincipalName+") and " + $ub.DisplayName+"("+$ub.UserPrincipalName+")  are in "+$mb[$i].DisplayName +"("+$mb[$i].ObjectID+")"
			}
		}
	}
}
```

## サンプルの出力結果
```
Joni Sherman(JoniS@xxx.OnMicrosoft.com) and Debra Berger(DebraB@xxx.OnMicrosoft.com)  are in Sales and Marketing(8cd8e24a-71da-4891-b5ce-b38b7905944d)
Joni Sherman(JoniS@xxx.OnMicrosoft.com) and Pradeep Gupta(PradeepG@xxx.OnMicrosoft.com)  are in Sales and Marketing(8cd8e24a-71da-4891-b5ce-b38b7905944d)
Irvin Sayers(IrvinS@xxx.OnMicrosoft.com) and Debra Berger(DebraB@xxx.OnMicrosoft.com)  are in Mark 8 Project Team(ac6d81e5-0f73-4127-9bde-228c4b9bc20f)
```
