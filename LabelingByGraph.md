# Graph API を利用して SharePoint Online のファイルに秘密度ラベルを付与する
SharePoint Online のファイルに Graph API を利用して秘密度ラベルを付与するサンプルです。
Graph API を用いた秘密度ラベルの付与では、1 操作につき、[$0.0018 の従量課金](https://learn.microsoft.com/ja-jp/graph/metered-api-list)が発生するため、Azure のサブスクリプション環境が必要となります。

## 事前準備
### アプリケーションの登録
1. Entra ID (https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps) でアプリケーションを登録する
3. 作成したアプリケーションの概要ページから、アプリケーション (クライアント) ID の値をコピーしておく。
4. 管理の API のアクセス許可で、アプリケーションの許可として Sites.ReadWrite.All の権限を与える
5. 付与した権限に対して管理者の同意を与える
6. 作成したアプリケーションの Client ID を用いて以下の URL の後ろのパラーメーターを書き換え   
7. 管理の証明書とシークレットから Client Secret を新規作成して値をコピーしておく

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
#環境変数
$app="先の手順 2 で取得したアプリケーション ID の値"
$sec="先の手順 6 で取得したアプリケーション ID の値"
$tenant="Azure PortalのEntra ID の概要ページで確認できるテナント ID"
$label="秘密度ラベルの GUID "

#ラベル付けしたいファイルがある SPO のサイトの指定 https:// 入れずに、FQDN の後とサイトの URL の後に : を入れることに注意
$sitePath="xxx.sharepoint.com:/sites/label:"
#ラベル付けしたいファイルがあるドキュメント ライブラリの名称
$listName="ドキュメント"
#ラベル付けしたいファイルの名前
$fileName="議事録1.docx"

#Graph にクライアント シークレットで接続
$sec2 = ConvertTo-SecureString -String $sec -AsPlainText -Force
$sec3 = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $app, $sec2
Connect-MgGraph -NoWelcome -ClientSecretCredential $sec3 -TenantId $tenant

#サイトを取得
$site=Get-MgSite -SiteId $sitePath

#リストを取得
$list=Get-MgSiteList -SiteId $site.Id -Filter "DisplayName eq '$listName'"

#全リスト アイテムを取得
$items=Get-MgSiteListItem -SiteId $site.Id -ListId $list.Id

#対象のファイルを取得
$enc=([System.Web.HttpUtility]::UrlEncode($fileName)).ToUpper()
$item=$items|?{$_.WebUrl.ToUpper().EndsWith($enc)}

#DriveItemとして取得しなおす
$file=Get-MgSiteListItemDriveItem -SiteId $site.Id -ListId $list.Id -ListItemId $item.Id

#パラメータを準備
$params = @{
  "sensitivityLabelId"=$label
  "assignmentMethod"="standard"
  "justificationText"="Labeled by Graph"
}

#ラベル付けを実施
Invoke-MgGraphRequest -Method "POST" -Uri $Uri -Body $params
```


