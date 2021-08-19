```
#監査対象のグループ
$GroupforAudit ="All Employees"
#監査対象のグループのメンバーが参加許可されているTeams チーム
$AllowedTeams=("Contoso Team", "Ask HR", "Operations")
#接続に利用するID
$connectionID="xxxx@xxxx.onmicrosoft.com"

Connect-AzureAD -AccountId $connectionID

#監視対象のグループの取得
$g=Get-AzureADGroup -SearchString $GroupforAudit

#許可されているTeams チームの取得
$AllowedTeamsGuid=@();
Foreach($t in $AllowedTeams){
	$r=Get-AzureADGroup -SearchString $t
	$AllowedTeamsGuid+=$r.ObjectId
}

#対象のユーザーの取得
$UsersforAudit=Get-AzureADGroupMember -ObjectId $g.ObjectId -All $true|Where-Object {$_.UserType -eq "Member"} 

Foreach($u in $UsersforAudit){
	#ユーザーのTeamsのメンバーシップを取得(簡易的に SPO と接続している Microsoft 365 Groups として判断)
	$membership=Get-AzureADUserMembership -ObjectId $u.ObjectID -All $true |Where-Object {$_.ProxyAddresses.Count -eq 2 -and $_.ProxyAddresses[0].StartsWith("SPO:")}

	Foreach($m in $membership){
		$index=$AllowedTeamsGuid.IndexOf($m.ObjectId)
    #許可されていないグループへの参加があれば、出力
		If($index -eq -1){$u.DisplayName+"("+$u.UserPrincipalName+") is in "+$m.DisplayName +"("+$m.ObjectID+")"
		}
	}
}![image](https://user-images.githubusercontent.com/66407692/130087380-6f966ff8-a3bb-4a28-9c50-603848a109ac.png)
```

#出力サンプル
```
MOD Administrator(admin@xxxxx.onmicrosoft.com) is in Sales and Marketing(8cd8e24a-71da-4891-b5ce-b38b7905944d)
MOD Administrator(admin@xxxxx.onmicrosoft.com) is in Mark 8 Project Team(ac6d81e5-0f73-4127-9bde-228c4b9bc20f)
MOD Administrator(admin@xxxxx.onmicrosoft.com) is in Retail(cc1265cc-842d-4dae-89d3-08caf2f5da77)
MOD Administrator(admin@xxxxx.onmicrosoft.com) is in Digital Initiative Public Relations(e88fe2fc-9a3e-48f0-8a37-78a3db263866)
MOD Administrator(admin@xxxxx.onmicrosoft.com) is in U.S. Sales(04a1f966-ff6f-4351-8418-23a28700528d)
MOD Administrator(admin@xxxxx.onmicrosoft.com) is in Contoso(09645b11-46ce-49bf-ad5d-53a9ac8feba5)
MOD Administrator(admin@xxxxx.onmicrosoft.com) is in SOC Team(e3d6cc62-a542-4563-b182-8c6558ba0172)
MOD Administrator(admin@xxxxx.onmicrosoft.com) is in 営業部サイト(f62f8742-ab3a-4684-9c22-eac6f440a1cd)
MOD Administrator(admin@xxxxx.onmicrosoft.com) is in Private Teams(04acab9e-4cc1-442b-90f6-9504b3a4151a)
MOD Administrator(admin@xxxxx.onmicrosoft.com) is in 営業部グループ(99b2bb02-781a-4836-990c-c4e7c7595c15)
Alex Wilber(AlexW@xxxxx.OnMicrosoft.com) is in CEO Connection(362fac39-7270-49aa-a7ef-0b3a8c9b2881)
Alex Wilber(AlexW@xxxxx.OnMicrosoft.com) is in Sales Best Practices(e08b3a74-c07a-414f-b401-e14a8706412e)
Alex Wilber(AlexW@xxxxx.OnMicrosoft.com) is in Sales and Marketing(8cd8e24a-71da-4891-b5ce-b38b7905944d)
Alex Wilber(AlexW@xxxxx.OnMicrosoft.com) is in Mark 8 Project Team(ac6d81e5-0f73-4127-9bde-228c4b9bc20f)
Alex Wilber(AlexW@xxxxx.OnMicrosoft.com) is in Retail(cc1265cc-842d-4dae-89d3-08caf2f5da77)
Alex Wilber(AlexW@xxxxx.OnMicrosoft.com) is in Digital Initiative Public Relations(e88fe2fc-9a3e-48f0-8a37-78a3db263866)
````
