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
#接続に利用するID / パスワード
$Id="xxx@xxx.onmicrosoft.com"
$Password = "xxxxx"
$OutputFolder=[System.Environment]::GetFolderPath("Desktop")+"\"

#Credentialの生成
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Connect-AzureAD -credential $Credential
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

#監視対象のグループを取得
$ga=Get-AzureADGroup -SearchString $GroupforAuditA
$gb=Get-AzureADGroup -SearchString $GroupforAuditB

#Group A のユーザーの取得
$UsersforAuditA=Get-AzureADGroupMember -ObjectId $ga.ObjectId -All $true|?{$_.UserType -eq "Member"} 

#Group B のユーザーの取得
$UsersforAuditB=Get-AzureADGroupMember -ObjectId $gb.ObjectId -All $true|?{$_.UserType -eq "Member"} 

#事前にGorup Bのメンバーの全メンバーシップを取得
$UserBTable=@()
$MembershipBTable=@()
Foreach($ub in $UsersforAuditB){
	$membershipB=Get-AzureADUserMembership -ObjectId $ub.ObjectID -All $true |?{$_.ObjectType -eq "Group"} |?{$TeamsGuid.IndexOf($_.ObjectId) -ne -1}
	$UserBTable+=$ub
	#2次元配列とするため , を入れる
	$MembershipBTable+=,$membershipB
}

$csv=@()
#Group A のメンバーと Group B のメンバーが共通に属する Teams チームを見つける
Foreach($ua in $UsersforAuditA){
	#Group A のメンバーのメンバーシップを取得
	$membershipA=Get-AzureADUserMembership -ObjectId $ua.ObjectID -All $true |?{$_.ObjectType -eq "Group"}|?{$TeamsGuid.IndexOf($_.ObjectId) -ne -1}
	Foreach($ma in $membershipA){
		Foreach($mb in $MembershipBTable){
			$i=$mb.IndexOf($ma)
			If($i -ne -1){
			$ub=$UserBTable[$MembershipBTable.IndexOf($mb)]
			if($ua.UserPrincipalName -eq $ub.UserPrincipalName) {continue}
			$line = New-Object PSObject | Select-Object UserAName, UserAUPN, UserBName, UserBUPN, TeamName, TeamGuid
			$line.UserAName=$ua.DisplayName
			$line.UserAUPN=$ua.UserPrincipalName
			$line.UserBName=$ub.DisplayName
			$line.UserBUPN=$ub.UserPrincipalName
			$line.TeamName=$mb[$i].DisplayName
			$line.TeamGuid=$mb[$i].ObjectID
			$csv+=$line
			}
		}
	}
}

#CSV出力
$csv|Export-csv -Path ($OutputFolder+"ConflictMembership"+(Get-Date -Format "yyyyMMdd-HHmm")+".csv") -Encoding UTF8 -NoTypeInformation
```

## サンプルの出力結果
```
"UserAName","UserAUPN","UserBName","UserBUPN","TeamName","TeamGuid"
"Joni Sherman","JoniS@xxxx.OnMicrosoft.com","Debra Berger","DebraB@xxxx.OnMicrosoft.com","Sales and Marketing","8cd8e24a-71da-4891-b5ce-b38b7905944d"
"Joni Sherman","JoniS@xxxx.OnMicrosoft.com","Pradeep Gupta","PradeepG@xxxx.OnMicrosoft.com","Sales and Marketing","8cd8e24a-71da-4891-b5ce-b38b7905944d"
"Grady Archie","GradyA@xxxx.OnMicrosoft.com","Pradeep Gupta","PradeepG@xxxx.OnMicrosoft.com","Mark 8 Project Team","ac6d81e5-0f73-4127-9bde-228c4b9bc20f"
"Grady Archie","GradyA@xxxx.OnMicrosoft.com","Megan Bowen","MeganB@xxxx.OnMicrosoft.com","Mark 8 Project Team","ac6d81e5-0f73-4127-9bde-228c4b9bc20f"
"Grady Archie","GradyA@xxxx.OnMicrosoft.com","Megan Bowen","MeganB@xxxx.OnMicrosoft.com","Contoso","09645b11-46ce-49bf-ad5d-53a9ac8feba5"
"Grady Archie","GradyA@xxxx.OnMicrosoft.com","Diego Siciliani","DiegoS@xxxx.OnMicrosoft.com","Contoso","09645b11-46ce-49bf-ad5d-53a9ac8feba5"
```
