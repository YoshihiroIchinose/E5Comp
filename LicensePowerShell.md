# サービス プランの割り当て状況に応じたセキュリティ グループのメンテナンス
ライセンスを細分化したアプリ単位となるサービス プランの割り当て状況に応じたセキュリティ グループのメンテナンスを実施する PowerShel スクリプトのサンプルです。
ここでは、"INTUNE_A"のサービス プランが割当たった社内ユーザーを、現在のグループ メンバシップとの差分を確認しながら、"With_INTUNE_A"というセキュリティ グループに登録し、割当たっていないユーザーを"Without_INTUNE_A"というセキュリティ グループに登録します。セキュリティ グループが作成されていない場合、セキュリティ グループも作成します。。なおライセンスの SKU や、各 SKU に含まれるサービス プランについては、[こちら](https://docs.microsoft.com/ja-jp/azure/active-directory/enterprise-users/licensing-service-plan-reference) に記載があります。

# 接続と ライセンス (SKU) とサービス プランの確認
```
$Id="xxxx@xxxx.onmicrosoft.com"
$Password = "xxxx"

#Credentialの生成
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Connect-AzureAD -Credential $Credential
$PlanName="INTUNE_A"

$skus=Get-AzureADSubscribedSku
$target=@()
$spid=""

#ServicePlan名から該当のSKU ID(複数ありうる)と、ServicePlanのIDを取得
foreach($sku in $skus){
  foreach($sp in $sku.ServicePlans){
	if($sp.ServicePlanName -eq $PlanName){
	 $target+=$sku.SkuId
	$spid=$sp.ServicePlanId
	}
  }
}
```
## ライセンス保有状況に応じた社内メンバーのグループのメンテナンス
```
#対象となるセキュリティ グループ名
$LicencedGroup="With_$PlanName"

#ライセンスがあるユーザーの取得
$licensedUsers=@()
$users=Get-AzureADUser -All $true -Filter "userType eq 'Member'"
foreach($u in $users){
	foreach($al in $u.AssignedLicenses){
	if($target.Contains($al.SkuId) -and !$al.DisabledPlans.Contains($spid)){
			$licensedUsers+=$u
			break
		}
	}
}

#ライセンスがあるユーザーのグループをメンテナンス
$lg=Get-AzureADGroup -Filter "DisplayName eq '$LicencedGroup'"
if($lg -eq $null){
	$lg=New-AzureADGroup -SecurityEnabled $true -MailEnabled $false -MailNickName $LicencedGroup -DisplayName $LicencedGroup
}

$mem=Get-AzureADGroupMember -All $true -ObjectId $lg.ObjectId
#グループからライセンスがなくなったユーザーを削除
foreach($m in $mem){
	if(!$licensedUsers.Contains($m)){
		Remove-AzureADGroupMember  -ObjectId $lg.ObjectId -MemberId $m.ObjectId
	}
}

#グループに登録されていないライセンスがあるユーザーを追加
foreach($lu in $licensedUsers){
	if($mem -eq $null -or !$mem.Contains($lu)){
		Add-AzureADGroupMember  -ObjectId $lg.ObjectId -RefObjectId $lu.ObjectId
		}
}
```
## ライセンスを保有していない社内メンバーのグループのメンテナンス
```
#対象となるセキュリティ グループ名
$UnLicencedGroup="Without_$PlanName"

#ライセンスがないユーザーの取得
$UsersWoL=@()
$users=Get-AzureADUser -All $true -Filter "userType eq 'Member'"
foreach($u in $users){
	$found=$false
	foreach($al in $u.AssignedLicenses){
	if($target.Contains($al.SkuId) -and !$al.DisabledPlans.Contains($spid)){
			$found=$true
			break
		}
	}
	if(!$found){$UsersWoL+=$u}
}
#ライセンスがないユーザーのグループをメンテナンス
$ug=Get-AzureADGroup -Filter "DisplayName eq '$UnLicencedGroup'"
if($ug -eq $null){
	$ug=New-AzureADGroup -SecurityEnabled $true -MailEnabled $false -MailNickName $UnLicencedGroup -DisplayName $UnLicencedGroup
}

$mem=Get-AzureADGroupMember -All $true -ObjectId $ug.ObjectId
#グループからライセンスが付与されたユーザーを削除
foreach($m in $mem){
	if(!$UsersWoL.Contains($m)){
		Remove-AzureADGroupMember  -ObjectId $ug.ObjectId -MemberId $m.ObjectId
	}
}

#グループにライセンスがなくなったユーザーを追加
foreach($u in $UsersWoL){
	if($mem -eq $null -or !$mem.Contains($u)){
		Add-AzureADGroupMember  -ObjectId $ug.ObjectId -RefObjectId $u.ObjectId
		}
}

```
