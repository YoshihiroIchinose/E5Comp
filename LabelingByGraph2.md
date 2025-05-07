# Graph API を利用してサイト管理者権限と全体管理者の同意で SharePoint Online のファイルに秘密度ラベルを付与する
SharePoint Online のファイルに Graph API を利用して秘密度ラベルを付与するサンプルです。アプリケーションに SPO サイト全体の権限を付与するのではなく、特定のサイトのみの権限を付与する場合のサンプルです。以下の表の No.2 のパターンとなります。なお、No.3 の構成は現状、従量課金の API では利用できません。[参考: 従量制課金 API の既知の制限](https://learn.microsoft.com/ja-jp/graph/metered-api-overview#known-limitations)

| No. | アクセス許可の種類 | アクセス許可 | 管理者の同意の必要性 | ラベル付け API での利用可否 | 
| --- | ---------------------- | --- | --- | --- |  
| 1 | アプリケーションの許可 | Sites.ReadWrite.All | 必要 | 可能 |
| 2 | アプリケーションの許可 | Sites.Selected | 必要 | 可能 |
| 3 | 委任されたアクセス許可 | Sites.Selected | 不要 | 不可 |

#### アプリケーションの許可
呼び出すユーザーの権限に関係なく、アプリケーションとして認証し、アプリケーション自身が直接付与された権限を持って、動作する形式   
#### 委任されたアクセス許可
各ユーザーで認証しつつ、アプリケーションがユーザーの代理として、ユーザーの同意もしくは管理者による組織全体の同意により、ユーザーが持つ権限の一部をアプリケーションが持って動作する形式   　

上記以外にも、特定の SPO サイトに対してアプリケーションに権限を付与するために、Microsoft Graph Command Line Tools(14d82eec-204b-4c2f-b7e8-296a70dab67e)に、ユーザーの代理としてサイトを管理できる Sites.FullControll.All の委任されたアクセス許可を管理者によって付与する必要があります。

Graph API を用いた秘密度ラベルの付与では、1 操作につき、[$0.0018 の従量課金](https://learn.microsoft.com/ja-jp/graph/metered-api-list)が発生するため、Azure のサブスクリプション環境が必要となります。

## 事前準備
### 1. アプリケーションの登録
1. Entra ID の[概要ページ](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/Overview) でテナントの ID をコピーしておく
2. Entra ID の[アプリの登録ページ](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps) でアプリケーションを登録する
3. 作成したアプリケーションの概要ページから、アプリケーション (クライアント) ID の値をコピーしておく
4. 管理の API のアクセス許可で、アクセス許可の追加から Microsoft Graph のアプリケーションの許可を選び、Sites.Selected の権限を与える
5. **付与した権限に対して管理者の同意を与えておく (管理者権限必要)** 
6. 管理の証明書とシークレットから Client Secret を新規作成して値をコピーしておく

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
**この方法は管理者権限必要**
他には、MPIPクライアントをインストールして、Get-FileLabel でラベル付けしたファイルを確認する方法がある
```PowerShell
$labelName="社外秘"
Connect-IPPSSession
$labels=Get-Label|?{$_.ContentType.IndexOf("File") -ne -1}
$labels|?{$_.DisplayName -eq $labelName}|select GUID
disconnect-ExchangeOnline
```

### 5. テナント管理者による MgGraph アプリでサイトの管理が行える権限を付与
**管理者権限必要**
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
$app="先の手順 1-3 で取得したアプリケーション ID の値"
$sec="先の手順 1-6 で取得したアプリケーションのクライアント シークレットの値"
$label="先の手順 4 で取得した秘密度ラベルの GUID "

#ラベル付けしたいファイルがある SPO のサイトの指定 https:// 入れずに、FQDN の後とサイトの URL の後に : を入れることに注意
$sitePath="xxx.sharepoint.com:/sites/Labeling2:"
#ラベル付けしたいファイルがあるドキュメント ライブラリの名称
$libraryName="ドキュメント"
#ラベル付けしたいファイルの名前
$fileName="議事録1.docx"

#Graph カスタム アプリケーションとして接続
$sec2 = ConvertTo-SecureString -String $sec -AsPlainText -Force
$sec3 = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $app, $sec2
Connect-MgGraph -NoWelcome -ClientSecretCredential $sec3 -TenantId $tenant

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

# <現状動作不可> ラベル付けを実施 (ラベルが反映されるまで、数分のラグがある)
#認証されたユーザーから委任された権限ではラベル付けは不可、アプリケーション固有の認証でないと従量課金の API は利用できない
Invoke-MgGraphRequest -Method "POST" -Uri $uri -Body $params
```
## 参考 URL
[Practical Graph: Assign Sensitivity Labels to SharePoint Online Files](https://practical365.com/assignsensitivitylabel-api/)
