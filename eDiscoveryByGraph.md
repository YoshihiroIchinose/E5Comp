# 本スクリプトは、Graph API を用いて eDiscovery のメールボックス検索を行うサンプルです
本スクリプトでは、Graph API を用いて eDiscvoery(Premium) のケース作成、カストーディアン(調査対象となるユーザー)の追加、
検索対象としてカストーディアンのメールボックスの指定、クレジット カード番号の SIT を見つける検索キーワードの登録、
レビュー セットの作成、レビューセットへの検索結果の反映までを自動化する例を示します。なお、既存のケースやレビュー セット等でも、
使いまわせるように、各種オブジェクトを作成した後、名前で各種オブジェクトを取得しなおすようなサンプルとしています。
また、カストーディアンの追加部分では、配列を用いているので複数調査対象への拡張もできるようにしています。
検索キーワードの部分では、EXO でも利用可能になった、機密情報の種類を使った検索をしていて、ここではクレジット カード番号の
SIT を用いています。

## 事前準備
管理権限で PowerShell を立ち上げ以下を一度実行
```
Install-Module Microsoft.Graph.Security
```

## スクリプト
```
#1. 適切な権限で接続
Connect-MgGraph -Scopes "eDiscovery.ReadWrite.All"

#2. ケース作成
$CaseName="ケース1 by Graph"
$params = @{
	displayName = $CaseName
	description = "Graphで作成するケース"
}
New-MgSecurityCaseEdiscoveryCase -BodyParameter $params

#2. ケース取得
$case=Get-MgSecurityCaseEdiscoveryCase|?{$_.DisplayName -eq $CaseName}

#3. カストーディアン追加
$emails=@("alexw@xxxx.onmicrosoft.com","admin@xxxx.onmicrosoft.com")
Foreach($email in $emails){
	$params = @{email = $email}
	New-MgSecurityCaseEdiscoveryCaseCustodian -EdiscoveryCaseId $case.id -BodyParameter $params
}

#3. カストーディアン取得
$custodians=Get-MgSecurityCaseEdiscoveryCaseCustodian -EdiscoveryCaseId $case.id

#4. メールボックス追加
Foreach($cust in $custodians){
	$params = @{email = $cust.Email
	includedSources = "mailbox"}
	New-MgSecurityCaseEdiscoveryCaseCustodianUserSource -EdiscoveryCaseId $case.id -EdiscoveryCustodianId $cust.Id -BodyParameter $params
}

#4. 対象メールボックス取得
$mailboxes=@()
Foreach($cust in $custodians){
	$mb=get-MgSecurityCaseEdiscoveryCaseCustodianUserSource -EdiscoveryCaseId $case.id -EdiscoveryCustodianId $cust.Id
	$baseURL="https://graph.microsoft.com/v1.0/security/cases/ediscoveryCases/"+$case.id+"/custodians/"
	$mailboxes+=$baseURL+$cust.id+"/userSources/"+$mb.Id
}

#5. 検索の登録
$searchname="Search by Graph"
$params = @{
	displayName = $searchname
	description = "Search by Graph"
	contentQuery = '((SensitiveType="50b8b56b-4ef8-44c2-a924-03374f5831ce|1..500|1..100"))'
	"custodianSources@odata.bind"=$mailboxes
}
New-MgBetaSecurityCaseEdiscoveryCaseSearch -EdiscoveryCaseId $case.id -BodyParameter $params

#5. 検索の取得
$search=Get-MgBetaSecurityCaseEdiscoveryCaseSearch -EdiscoveryCaseId $case.id|?{$_.DisplayName -eq $searchname}

#6. レビューセット作成
$ReviewsetName="Reviewset1"
$params = @{
	displayName = $ReviewsetName
}

#6. レビューセット取得
New-MgSecurityCaseEdiscoveryCaseReviewSet -EdiscoveryCaseId $case.id -BodyParameter $params
$reviewset=MgSecurityCaseEdiscoveryCaseReviewSet -EdiscoveryCaseId $case.id|?{$_.DisplayName -eq $ReviewsetName}

#7. レビューセットに検索結果を反映
$params = @{
	search = @{
		id = $search.id
	}
	additionalDataOptions = "linkedFiles"
}
Add-MgSecurityCaseEdiscoveryCaseReviewSetToReviewSet -EdiscoveryCaseId $case.id -EdiscoveryReviewSetId $reviewset.id -BodyParameter $params
```
