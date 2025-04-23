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
# 変数
$CaseName="ケース1 by Graph"
$SearchName="検索1 by Graph"
$ReviewsetName="レビュー セット1"
$emails=@("alexw@xxxx.onmicrosoft.com","admin@xxxx.onmicrosoft.com")
$query='(SensitiveType="50b8b56b-4ef8-44c2-a924-03374f5831ce|1..500|1..100") AND sent>=2025-01-01 AND sent<=2025-03-31'


#1. 適切な権限で接続
Connect-MgGraph -Scopes "eDiscovery.ReadWrite.All"

#2-1. ケース作成
New-MgSecurityCaseEdiscoveryCase -DisplayName $CaseName

#2-2. ケース取得
$case=Get-MgSecurityCaseEdiscoveryCase -Filter "DisplayName eq '$CaseName'"

#3-1. カストーディアン追加
Foreach($email in $emails){
	New-MgSecurityCaseEdiscoveryCaseCustodian -EdiscoveryCaseId $case.id -Email $email
	}

#3-2. カストーディアン取得
$custodians=Get-MgSecurityCaseEdiscoveryCaseCustodian -EdiscoveryCaseId $case.id

#4-1. メールボックス追加
Foreach($cust in $custodians){
	New-MgSecurityCaseEdiscoveryCaseCustodianUserSource -EdiscoveryCaseId $case.id -EdiscoveryCustodianId $cust.Id -Email $cust.Email -IncludedSources "mailbox"
}

#4-2. 対象メールボックス取得
$mailboxes=@()
Foreach($cust in $custodians){
	$mb=get-MgSecurityCaseEdiscoveryCaseCustodianUserSource -EdiscoveryCaseId $case.id -EdiscoveryCustodianId $cust.Id
	$baseURL="https://graph.microsoft.com/v1.0/security/cases/ediscoveryCases/"+$case.id+"/custodians/"
	$mailboxes+=$baseURL+$cust.id+"/userSources/"+$mb.Id
}

#5-1. 検索の登録
$params = @{
	displayName = $SearchName
	contentQuery = $query
	"custodianSources@odata.bind"=$mailboxes
}
New-MgSecurityCaseEdiscoveryCaseSearch -EdiscoveryCaseId $case.id -BodyParameter $params

#5-2. 検索の取得
$search=Get-MgSecurityCaseEdiscoveryCaseSearch -EdiscoveryCaseId $case.id|?{$_.DisplayName -eq $searchName}

#6-1. レビューセット作成
New-MgSecurityCaseEdiscoveryCaseReviewSet -EdiscoveryCaseId $case.id -DisplayName $ReviewsetName

#6-2. レビューセット取得
$reviewset=MgSecurityCaseEdiscoveryCaseReviewSet -EdiscoveryCaseId $case.id -Filter "displayName eq '$ReviewsetName'"

#7-1. レビューセットに検索結果を反映
$params = @{
	search = @{
		id = $search.id
	}
	additionalDataOptions = "linkedFiles"
}
#この部分だけBetaのGraph APIが必要かも
Add-MgBetaSecurityCaseEdiscoveryCaseReviewSetToReviewSet -EdiscoveryCaseId $case.id -EdiscoveryReviewSetId $reviewset.id -BodyParameter $params
```
