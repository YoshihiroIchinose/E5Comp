# Graph API を利用してサイト管理者権限と全体管理者の同意で SharePoint Online のファイルに秘密度ラベルを付与する
SharePoint Online のファイルに Graph API を利用して秘密度ラベルを付与するサンプルです。権限付与時の組織の同意部分を除いて、サイト管理者の権限で動作するものです。
Graph API を用いた秘密度ラベルの付与では、1 操作につき、[$0.0018 の従量課金](https://learn.microsoft.com/ja-jp/graph/metered-api-list)が発生するため、Azure のサブスクリプション環境が必要となります。

## 事前準備
### 1. アプリケーションの登録
1. Entra ID の[概要ページ](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/Overview) でテナントの ID をコピーしておく
2. Entra ID の[アプリの登録ページ](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps) でアプリケーションを登録する
3. 作成したアプリケーションの概要ページから、アプリケーション (クライアント) ID の値をコピーしておく
4. 管理の API のアクセス許可で、アクセス許可の追加から Microsoft Graph の委任されたアクセス許可を選び、Sites.Selected の権限を与える
5. 認証ページのプラットフォーム構成でプラットフォームを追加から「モバイル アプリケーションとデスクトップ アプリケーション」を選択し、「http://localhost」を入力し構成ボタンを押す

### 2. 従量課金の API の有効化
1. サブスクリプションを保有しているアカウントで Azure Portal にアクセスし、右上のメニューから Azure Cloud Shell を PowerShell のセッションで起動する
2. 以下のコマンドを実行する
```
$app="先の手順 1-2 で取得したアプリケーション ID の値"
$rgName="LabelingByGraph"
$name="LabelingAccounts2"
az group create --name $rgName --location westus
az resource create --resource-group $rgName --name $name --resource-type Microsoft.GraphServices/accounts --properties "{""appId"": ""$app""}" --location Global
```

### 3. スクリプト環境の準備
PowerShell を端末の管理権限で立ち上げて事前に一度以下を実行
```PowerShell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module -Name ExchangeOnlineManagement
Install-Module -Name Microsoft.Graph
```

### 4. 秘密度ラベルの GUID の把握
```PowerShell
$labelName="社外秘"
Connect-IPPSSession
$labels=Get-Label|?{$_.ContentType.IndexOf("File") -ne -1}
$labels|?{$_.DisplayName -eq $labelName}|select GUID
disconnect-ExchangeOnline
```

### 5. テナント管理者による MgGraph アプリでサイトの管理が行える権限を付与
PowerShell から以下を実行
```PowerShell
# 以下を実行した際、テナント管理者でサインインして、「組織の代理として同意する」にチェックを入れて承諾する
Connect-MgGraph -NoWelcome -Scopes "Sites.FullControl.All"
Disconnect-MgGraph
```

### 6. サイト管理者から MgGraph を通じて、ラベル付けアプリに Read & Write 権限を付与する
PowerShell から以下を実行
```PowerShell
#変数
$app="先の手順 1-2 で取得したアプリケーション ID の値"
$appName="ラベル付けアプリの名称"

#ラベル付けしたいファイルがある SPO のサイトの指定 https:// 入れずに、FQDN の後とサイトの URL の後に : を入れることに注意
$sitePath="xxx.sharepoint.com:/sites/Labeling2:"

# 以下を実行した際、サイト管理者でサインインする
Connect-MgGraph -NoWelcome -Scopes "Sites.FullControl.All"

$site=Get-MgSite -SiteId $sitePath

#ラベル付けアプリに権限付与する
$params = @{
　roles = @("write","read")
　grantedToIdentities= @(
　  @{application = @{
		  id = $app
		  displayName =$appName
		  }
	  }
  )
}
New-MgSitePermission -SiteId $site.Id -BodyParameter $params
Disconnect-MgGraph
```

## スクリプト　サンプル
### Graph API による単一のファイルへのラベル付け
```PowerShell
#環境変数
$tenant="先の手順 1-1 で取得したテナント ID"
$app="先の手順 1-2 で取得したアプリケーション ID の値"
$sec="先の手順 1-6 で取得したアプリケーション ID の値"
$label="先の手順 4 で取得した秘密度ラベルの GUID "

#ラベル付けしたいファイルがある SPO のサイトの指定 https:// 入れずに、FQDN の後とサイトの URL の後に : を入れることに注意
$sitePath="xxx.sharepoint.com:/sites/Labeling2:"
#ラベル付けしたいファイルがあるドキュメント ライブラリの名称
$libraryName="ドキュメント"
#ラベル付けしたいファイルの名前
$fileName="議事録1.docx"

#Graph にクライアント シークレットで接続
Connect-MgGraph -ClientId $app -TenantId $tenant -Scopes "Sites.Selected" -NoWelcome

#サイトを取得
$site=Get-MgSite -SiteId $sitePath

#Drive を取得
$drive=Get-MgSiteDrive -SiteId $site.id -Filter "Name eq '$libraryName'"

#Fileを取得
$file=Get-MgDriveItemChild -DriveId $drive.Id -DriveItemId "root" -Filter "Name eq '$fileName'"

#パラメータを準備
$params = @{
  "sensitivityLabelId"=$label
  "assignmentMethod"="standard"
  "justificationText"="Labeled by Graph"
}

#対象となるファイルを URI で指定
$uri = ("https://graph.microsoft.com/v1.0/sites/{0}/drives/{1}/items/{2}/assignSensitivityLabel" -f $site.Id,$drive.Id,$file.Id)

#ラベル付けを実施 (ラベルが反映されるまで、数分のラグがある)
Invoke-MgGraphRequest -Method "POST" -Uri $uri -Body $params
```
## 参考 URL
[Practical Graph: Assign Sensitivity Labels to SharePoint Online Files](https://practical365.com/assignsensitivitylabel-api/)
