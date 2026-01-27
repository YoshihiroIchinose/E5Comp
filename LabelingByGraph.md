# Graph API を利用して SharePoint Online のファイルに秘密度ラベルを付与する
SharePoint Online のファイルに Graph API を利用して秘密度ラベルを付与するサンプルです。
Graph API を用いた秘密度ラベルの付与では、1 操作につき、[$0.00185 の従量課金](https://learn.microsoft.com/ja-jp/graph/metered-api-list)が発生するため、Azure のサブスクリプション環境が必要となります。

## 事前準備
### 1. アプリケーションの登録
1. Entra ID の[概要ページ](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/Overview) でテナントの ID をコピーしておく
2. Entra ID の[アプリの登録ページ](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps) でアプリケーションを登録する
3. 作成したアプリケーションの概要ページから、アプリケーション (クライアント) ID の値をコピーしておく
4. 管理の API のアクセス許可で、アプリケーションの許可として Sites.ReadWrite.All の権限を与える
5. 付与した権限に対して管理者の同意を与える
6. 管理の証明書とシークレットからクライアント シークレットを新規作成して値をコピーしておく

アプリケーションに SPO サイト全体ではなく、特定サイトの権限だけを付与したい場合は、以下の[こちら](https://github.com/YoshihiroIchinose/E5Comp/blob/main/LabelingByGraph2.md)を参照のこと。

### 2. 従量課金の API の有効化
1. Azure Portal にアクセスし、右上のメニューから Azure Cloud Shell を PowerShell のセッションで起動する
2. 以下のコマンドを実行する
```
$app="先の手順 1-2 で取得したアプリケーション ID の値"
$rgName="LabelingByGraph"
$name="LabelingAccounts"
Register-AzResourceProvider -ProviderNamespace Microsoft.GraphServices
az group create --name $rgName --location westus
az resource create --resource-group $rgName --name $name --resource-type Microsoft.GraphServices/accounts --properties "{""appId"": ""$app""}" --location Global
```

### 3. スクリプト環境の準備
PowerShell を管理権限で立ち上げて事前に一度以下を実行
```
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module -Name ExchangeOnlineManagement
Install-Module -Name Microsoft.Graph
```

### 4. 秘密度ラベルの GUID の把握
```
$labelName="社外秘"
Connect-IPPSSession
$labels=Get-Label|?{$_.ContentType.IndexOf("File") -ne -1}
$labels|?{$_.DisplayName -eq $labelName}|select GUID
disconnect-ExchangeOnline
```

## スクリプト　サンプル
### Graph API による単一のファイルへのラベル付け
```PowerShell
#環境変数
$tenant="先の手順 1-1 で取得したテナント ID"
$app="先の手順 1-2 で取得したアプリケーション ID の値"
$sec="先の手順 1-6 で取得したアプリケーションのクライアント シークレットの値"
$label="先の手順 4 で取得した秘密度ラベルの GUID "

#ラベル付けしたいファイルがある SPO のサイトの指定 https:// 入れずに、"FQDN の後" と "サイトの URL" の後に : を入れることに注意
$sitePath="xxx.sharepoint.com:/sites/testsite01:"
#ラベル付けしたいファイルがあるドキュメント ライブラリの名称
$libraryName="ドキュメント"
#ラベル付けしたいファイルの名前
$fileName="議事録1.docx"

#Graph にクライアント シークレットで接続
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

#ラベル付けを実施 (ラベルが反映されるまで、数分のラグがある)
Invoke-MgGraphRequest -Method "POST" -Uri $uri -Body $params
```
### Graph API による特定フォルダ直下の複数ファイルへのラベル付け
優先度の高い低いに関わらず、既存ラベルも置き換える点に注意
```PowerShell
#環境変数
$tenant="先の手順 1-1 で取得したテナント ID"
$app="先の手順 1-2 で取得したアプリケーション ID の値"
$sec="先の手順 1-6 で取得したアプリケーションのクライアント シークレットの値"
$label="先の手順 4 で取得した秘密度ラベルの GUID "

#ラベル付けしたいファイルがある SPO のサイトの指定 https:// 入れずに、"FQDN の後" と" サイトの URL" の後に : を入れることに注意
$sitePath="xxx.sharepoint.com:/sites/testsite01:"
#ラベル付けしたいファイルがあるドキュメント ライブラリの名称
$libraryName="ドキュメント"
#ラベル付けしたいライブラリ内のフォルダのパス
$folder="社外秘保護"
#階層の場合
#$folder="社外秘保護/暗号化"

#Graph にクライアント シークレットで接続
$sec2 = ConvertTo-SecureString -String $sec -AsPlainText -Force
$sec3 = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $app, $sec2
Connect-MgGraph -NoWelcome -ClientSecretCredential $sec3 -TenantId $tenant

#サイトを取得
$site=Get-MgSite -SiteId $sitePath

#Drive を取得
$drive=Get-MgSiteDrive -SiteId $site.id -Filter "Name eq '$libraryName'"

#特定のフォルダ内のファイルを取得
$files=Get-MgDriveItemChild -DriveId $drive.Id -DriveItemId ("root:/"+$folder+":")

#パラメータを準備
$params = @{
  "sensitivityLabelId"=$label
  "assignmentMethod"="standard"
  "justificationText"="Labeled by Graph"
}

#対応ファイルに限定
$supported=@("docx","pptx","xlsx","pdf")

Foreach($file in $files){
	If(!$supported.contains($file.Name.split(".")[-1])){
		continue
  }
	$base ="https://graph.microsoft.com/v1.0/sites/$($site.Id)/drives/$($drive.Id)/items/"
	$uri=$base+$file.Id+"/extractSensitivityLabels"
	$l=Invoke-MgGraphRequest -Method "POST" -Uri $uri
	If($l.labels.sensitivityLabelId -eq $label){
      "ラベル付与済みのため'"+$file.Name +"'のラベル付けはスキップ"
  }
  else {
      "'"+$file.Name +"'へのラベル付けを実施"
      $uri=$base+$file.Id+"/assignSensitivityLabel"
      Invoke-MgGraphRequest -Method "POST" -Uri $uri -Body $params
	}
}
```
### Graph API による特定フォルダ直下の複数ファイルへのラベル付けを並列処理で行う
優先度の高い低いに関わらず、既存ラベルも置き換える点に注意
```PowerShell
#環境変数
$tenant="先の手順 1-1 で取得したテナント ID"
$app="先の手順 1-2 で取得したアプリケーション ID の値"
$sec="先の手順 1-6 で取得したアプリケーションのクライアント シークレットの値"
$label="先の手順 4 で取得した秘密度ラベルの GUID "

#ラベル付けしたいファイルがある SPO のサイトの指定 https:// 入れずに、"FQDN の後" と" サイトの URL" の後に : を入れることに注意
$sitePath="xxx.sharepoint.com:/sites/testsite01:"
#ラベル付けしたいファイルがあるドキュメント ライブラリの名称
$libraryName="ドキュメント"
#ラベル付けしたいライブラリ内のフォルダのパス
$folder="テスト用フォルダ"

#トークン取得用の Header
$body = @{  
    client_Id  = $app
     client_secret = $sec
    scope = 'https://graph.microsoft.com/.default'
    grant_type = 'client_credentials'
}

#トークン取得
$uri="https://login.microsoftonline.com/$tenant/oauth2/V2.0/token"
$tokenRequest = Invoke-WebRequest -Method Post -Uri $uri -Body $body
$token = ($tokenRequest.Content | ConvertFrom-Json).access_token

#Graph にトークンで接続
$sec2 = ConvertTo-SecureString -String $token -AsPlainText -Force
Connect-MgGraph -AccessToken $sec2  -NoWelcome

#サイトを取得
$site=Get-MgSite -SiteId $sitePath

#Drive を取得
$drive=Get-MgSiteDrive -SiteId $site.id -Filter "Name eq '$libraryName'"

#特定のフォルダ内のファイルを取得
$files=Get-MgDriveItemChild -DriveId $drive.Id -DriveItemId ("root:/"+$folder+":")

#ラベル付けの設定
$params = @{
	"sensitivityLabelId"=$label
  	"assignmentMethod"="standard"
  	"justificationText"="Labeled by Graph"
}

#並列処理用のワークフローを定義 既存 AccessToken を再利用
Workflow Label-Files(){
	param($files,$base,$sec2,$label,$params)
	Foreach -parallel ($file in $files){
			Connect-MgGraph -AccessToken $sec2  -NoWelcome
			$uri=$base+$file.Id+"/extractSensitivityLabels"
			$l=Invoke-MgGraphRequest -Method "POST" -Uri $uri
			If($l.labels.sensitivityLabelId -eq $label){
			"ラベル付与済みのため'"+$file.Name +"'のラベル付けはスキップ"
		  }
	  else {
	              	"'"+$file.Name +"'へのラベル付けを実施"
	             	$uri=$base+$file.Id+"/assignSensitivityLabel"
	             	Invoke-MgGraphRequest -Method "POST" -Uri $uri -Body $params
		}
	}
}

#対応拡張子に限定
$supported=@("docx","pptx","xlsx","pdf")
$files=$files|?{$supported.contains($_.Name.split(".")[-1])}

#並列処理でラベル付けを実施
$base ="https://graph.microsoft.com/v1.0/sites/$($site.Id)/drives/$($drive.Id)/items/"
Label-Files $files $base $sec2 $label $params
```
### Graph API による特定フォルダ配下の複数ファイルを再帰的に検出してラベル付けを並列処理で行う
優先度の高い低いに関わらず、既存ラベルも置き換える点に注意
```PowerShell
#環境変数
$tenant="先の手順 1-1 で取得したテナント ID"
$app="先の手順 1-2 で取得したアプリケーション ID の値"
$sec="先の手順 1-6 で取得したアプリケーションのクライアント シークレットの値"
$label="先の手順 4 で取得した秘密度ラベルの GUID "

#ラベル付けしたいファイルがある SPO のサイトの指定 https:// 入れずに、"FQDN の後" と "サイトの URL" の後に : を入れることに注意
$sitePath="xxx.sharepoint.com:/sites/testsite01:"
#ラベル付けしたいファイルがあるドキュメント ライブラリの名称
$libraryName="ドキュメント"
#ラベル付けしたいライブラリ内のフォルダのパス
$folder="テスト用フォルダ"

#トークン取得用の Header
$body = @{  
    client_Id  = $app
    client_secret = $sec
    scope = 'https://graph.microsoft.com/.default'
    grant_type = 'client_credentials'
}

#トークン取得
$uri="https://login.microsoftonline.com/$tenant/oauth2/V2.0/token"
$tokenRequest = Invoke-WebRequest -Method Post -Uri $uri -Body $body
$token = ($tokenRequest.Content | ConvertFrom-Json).access_token

#Graph にトークンで接続
$sec2 = ConvertTo-SecureString -String $token -AsPlainText -Force
Connect-MgGraph -AccessToken $sec2 -NoWelcome

#サイトを取得
$site=Get-MgSite -SiteId $sitePath

#Drive を取得
$drive=Get-MgSiteDrive -SiteId $site.id -Filter "Name eq '$libraryName'"

#対応拡張子に限定
$supported=@("docx","pptx","xlsx","pdf")

#対象ファイルの連想配列
$files=@{}

#再帰的にフォルダを掘る関数
Function Get-FolderItems (){
	param($folder)
	$current="root:/"+$folder+":"
	"フォルダの参照:" + $current
	#指定フォルダ内のアイテムを取得
	$items=Get-MgDriveItemChild -DriveId $drive.Id -DriveItemId ($current)
	Foreach($item in $items){
		#アイテムがラベル付け対象のファイルの場合
		If($item.Folder.ChildCount -eq $null -and $supported.contains($item.Name.split(".")[-1])){
			$files[$item.Id]=$item.Name
		}
		#アイテムがフォルダの場合
		ElseIf($item.Folder.ChildCount -ge 1){
			Get-FolderItems ($folder+"/"+$item.Name)
		}
	}
}

#指定したフォルダ配下のファイルを再帰的に検出
Get-FolderItems $folder

#ラベル付けの設定
$params = @{
	"sensitivityLabelId"=$label
  	"assignmentMethod"="standard"
  	"justificationText"="Labeled by Graph"
}

#並列処理用のワークフローを定義 既存 AccessToken を再利用
Workflow Label-Files(){
	param($files,$base,$sec2,$label,$params)
	Foreach -parallel ($key in $files.keys){
			Connect-MgGraph -AccessToken $sec2  -NoWelcome
			$uri=$base+$key+"/extractSensitivityLabels"
			$l=Invoke-MgGraphRequest -Method "POST" -Uri $uri
			If($l.labels.sensitivityLabelId -eq $label){
			"ラベル付与済みのため'"+$files[$key] +"'のラベル付けはスキップ"
		  }
	  else {
	              	"'"+$files[$key]+"'へのラベル付けを実施"
	             	$uri=$base+$key+"/assignSensitivityLabel"
	             	Invoke-MgGraphRequest -Method "POST" -Uri $uri -Body $params
		}
	}
}


#並列処理でラベル付けを実施
$base ="https://graph.microsoft.com/v1.0/sites/$($site.Id)/drives/$($drive.Id)/items/"
Label-Files $files $base $sec2 $label $params
```

## 参考 URL
[Practical Graph: Assign Sensitivity Labels to SharePoint Online Files](https://practical365.com/assignsensitivitylabel-api/)

