# Graph API を利用して SharePoint Online のファイルに秘密度ラベルを付与する
SharePoint Online のファイルに Graph API を利用して秘密度ラベルを付与するサンプルです。
Graph API を用いた秘密度ラベルの付与では、1 操作につき、[$0.0018 の従量課金](https://learn.microsoft.com/ja-jp/graph/metered-api-list)が発生するため、Azure のサブスクリプション環境が必要となります。


## 事前準備
### アプリケーションの登録
1. Azure AD (https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps) でアプリケーションを登録する
2. 作成したアプリケーションの概要ページから、アプリケーション (クライアント) ID の値をコピーしておく。
3. 管理の API のアクセス許可で、アプリケーションの許可として Sites.ReadWrite.All の権限を与える
4. 付与した権限に対して管理者の同意を与える
5. 作成したアプリケーションの Client ID を用いて以下の URL の後ろのパラーメーターを書き換え   
6. 管理の証明書とシークレットから Client Secret を新規作成して値をコピーしておく

### 従量課金の API の有効化
1. Azure Portal にアクセスし、右上のメニューから Azure Cloud Shell を PowerShell のセッションで起動する
2. 以下のコマンドを実行する
```
$app="先の手順 2 で取得したアプリケーション ID の値"
$rgName="LabelingByGraph"
$name="LabelingAccounts"
az group create --name $rgName --location westus
az resource create --resource-group $rgName --name $name --resource-type Microsoft.GraphServices/accounts --properties "{""appId"": ""$app""}" --location Global
```

### スクリプト環境の準備
PowerShell を管理権限で立ち上げて事前に一度以下を実行
```
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module -Name ExchangeOnlineManagement
Install-Module Microsoft.Graph
```

### 秘密度ラベルの GUID の把握
```
Connect-IPPSSession
$labels=Get-Label|?{$_.ContentType.IndexOf("File") -ne -1}
$labels|?{$_.DisplayName -eq "All Employees"}|select GUID
disconnect-ExchangeOnline
```

## Graph API によるラベル付け
```

```


