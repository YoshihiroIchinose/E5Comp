# Office 365 管理用の PowerShell スクリプトでの資格情報の管理手法について
各種 Office 365 管理用のスクリプトの中で、無人で定期実行したい場合には、対話的に資格情報を入力できないため資格情報を何等かの形で事前に持たせておく必要があります。
もちろんスクリプトの中に、ID およびパスワードを埋め込んで、利用することが最も簡単ですが、スクリプトの漏えいなどあれば、簡単に管理者権限を持つ ID が盗まれてしまうことになります。
そのため、今回そういったスクリプトの用途に利用可能な資格の保管・呼び出し方法についてまとめておきます。

|  手法  |  概要  | 利点 | 注意点 |
| ---- | ---- | ---- | ---- |
|  ① AppId と証明書の利用  |  Azure AD に管理接続用のアプリケーションを登録し、適切な権限を付与し、別途作成した証明書を紐づけ。各種スクリプトからは、そのアプリケーションの AppId と秘密鍵付きの証明書を利用して接続する。 | スクリプトの中に資格情報を記載せず、スクリプトと資格情報を分けて管理ができる。また管理者アカウントも用いないため、権限を制限しやすい。 |  対応しているサービスが限られている。証明書の管理が必要。秘密鍵付きの証明書のパスワードの管理も考慮が必要。|
|  ② SecureString をスクリプトに埋め込む  | 端末上のユーザー プロファイル固有の暗号化鍵を使って、事前に暗号化したパスワードを用意しておき、それをスクリプトに直接記載する。| 暗号化されたパスワードは、同一端末、同一ユーザー プロファイルの中でしか有効ではないため、スクリプト漏えい時の再利用性を制限可能。| 一か所でまとめて資格情報を管理できない。|
|  ③ OS の資格情報マネージャーを利用  | Windows OS の資格情報マネージャーに資格情報を登録しておき、スクリプトから取得。| スクリプトの中に資格情報を記載せず、スクリプトと資格情報を分けて管理ができる。OS の UI からも更新・バックアップが可能。| OS 環境が乗っ取られた場合には、攻撃者に抜き出されやすい。 |
|  ④ Azure Automation の資格情報管理機能を利用  | Azure Automation を使った PowerShell スクリプトの定期実行の場合、Automation アカウントに登録しておいた資格情報を呼び出して利用可能。 | スクリプトの中に資格情報を記載せず、スクリプトと資格情報を分けて管理ができる。Azure の UI からも更新・バックアップが可能。| Azure 環境が乗っ取られた場合には、攻撃者に抜き出されやすい。 |

# ① AppId と証明書の利用
Azure AD に管理接続用のアプリケーションを登録し、適切な権限を付与し、別途作成した証明書を紐づけ。各種スクリプトからは、そのアプリケーションの AppId と秘密鍵付きの証明書を利用して接続する。
## サービスの対応状況
|  サービス |  PowerShell コマンドレット | 証明書認証 | (参考 Azure AD アクセス トークンでの認証) |
| ---- | ---- | ---- | ---- |
| Exchange | Connect-ExchangeOnline | 2.0.3 2020/09/21 で対応| 未対応 |
| Compliance | Connect-IPPSSession | 2.0.6-Preview5 2022/03/17 で対応 <br> (Install-Module -Name ExchangeOnlineManagement -AllowPrerelease を実行のこと)| 未対応 |
| Azure AD | Connect-AzureAD | 対応済み| 対応 |
| SharePoint | Connect-SPOSerivce| 未対応 | 未対応 |
| Teams| Connect-MicrosoftTeams | 未対応 | 対応 |

## Azure AD 上でのアプリケーションの登録
こちらの[ガイド](https://docs.microsoft.com/ja-jp/powershell/exchange/app-only-auth-powershell-v2?view=exchange-ps)に従って、アプリケーションの登録、必要な権限の付与、自己署名証明書の作成、証明書の紐づけを実施のこと。

## 証明書を用いた接続
````
# 秘密鍵付き証明書 .pfx のパスワード(こちらも②や③と同じ管理をしても良いかもしれない)
$Password = "xxxx"
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force

Connect-IPPSSession -AppId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" -CertificateFilePath "C:\Cert\PowerShell.pfx" -Organization "xxxx.onmicrosoft.com" -CertificatePassword $SecPass
````

# ② SecureString をスクリプトに埋め込む
端末上のユーザー プロファイル固有の暗号化鍵を使って、事前に暗号化したパスワードを用意しておき、それをスクリプトに直接記載する方法。
## 資格情報の事前生成
````
$Password = "xxxx"
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
# 以下のコマンドでの出力を文字列としてコピーしておく
ConvertFrom-Securestring -securestring $SecPass
````
## 暗号化された資格情報のスクリプトへの埋め込みと利用
````
$Id="xxxx@xxxx.onmicrosoft.com"
#事前に前のステップで暗号化した文字列をスクリプトの中に埋め込んでおく
$EncPass="01000000d08c9ddf0115d1118c7a00c04fc297eb01000000513c089145a37b4894b08f5f410292f100000000020000000000106600000001000020000000daaf3caa201856f67fd45ff2fb215c6d3855620ae57924588b709cd1f0c10296000000000e8000000002000020000000ed0981d67f1d1e2a396bb609a8d387246ca802c745dd9950d7992d16090523d51000000058d8a486dedc1835f25a89bfbd8d110940000000e8fd09690f1f1a3a8d4ff6bb522a455801e5514a8944225231afaf2aeaabb545924b3aa4d2c2cda3d594ee1b800afcdd84e3ac04d3189cb3c13dc9cee8b79306"
#Credentialの生成
$SecPass= convertto-securestring -string $EncPass
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Connect-ExchangeOnline -credential $Credential
````

# ③ OS の資格情報マネージャーを利用
Windows OS の資格情報マネージャーに資格情報を登録しておき、スクリプトから取得する方法。
## 事前準備
管理者 PowerShell で一度以下を実行。
````
Install-Module CredentialManager
````
## 資格情報の事前登録
````
$Id="xxxx@xxxx.onmicrosoft.com"
$Password = "xxxx"
New-StoredCredential -Target "Office 365" -Username $Id -Password $Password -Persist Enterprise
````
## スクリプトでの資格情報の取得と利用
````
$Credential=Get-StoredCredential -Target  "Office 365"
Connect-ExchangeOnline -credential $SecPass
````
## UI へのアクセス
Windows の資格情報マネージャーは、コントロール パネルから起動するか、Windows キー + R の「ファイル名を指定して実行」から以下のコマンドを入力して起動可能。
````
control /name Microsoft.CredentialManager 
````

# ④ Azure Automation の資格情報管理機能を利用
Azure Automation を使った PowerShell スクリプトの定期実行の場合、Automation アカウントに登録しておいた資格情報を呼び出して利用可能。別途、Automation にて ExchangeOnlineManagement などのモジュールの読み込み設定をしておき、Automation アカウントの資格情報にて、"Office 365" の名称等で、ID/パスワードを設定しておくこと。
## スクリプトでの資格情報の取得と利用
````
$Credential = Get-AutomationPSCredential -Name "Office 365"
Connect-ExchangeOnline -credential $Credential
````

