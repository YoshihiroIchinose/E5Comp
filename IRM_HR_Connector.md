# Insider Risk Management の HR Connector に関連したスクリプト
## Azure Storage にアップロードされた CSV ファイルを、Azure Automation のスクリプトを通じて IRM の HR Connector に取り込む
1 つめのサンプルは、Azure Storage にアップロードされた CSV ファイルを、Azure Automatino のスクリプトを通じて IRM の HR Connector に取り込むスクリプトです。定期的に HR からの CSV ファイルを Azure Storage にアップロードしさえすれば、オンプレミスの常時稼働サーバーなしに Azure Automation で　HR データの IRM への取り込みをスケジュール化できます。

### 事前準備
1. Azure 上に Storage アカウントを作成し、そこにファイル シェア "hrdata" を作成する
2. 上記ファイル シェアに、"HR_Resignation.txt"　というファイル名で、 IRM HR Connector で取り込む CSV ファイルを、BOM 付き UTF-8 の形式でアップロードしておく
3. [こちら](https://github.com/microsoft/m365-compliance-connector-sample-scripts/blob/main/sample_script.ps1)で公開されているスクリプトをメモ帳などに張り付け、BOM 付き UTF-8 の形式で "HRConnector.ps1" という名称で保存する
4. 上記ファイルを、1 の "hrdata" のファイル シェアにアップロードする
6. Azure Automation アカウントを作成する
7. Azure Automation アカウントで、1.のファイル シェアへの接続キーを暗号化された文字列の変数として "storageKey" という名称で保存する
8. Azure Automation アカウントで、IRM のコネクタ用に Azure AD で登録したアプリケーションの Client Secret を暗号化された文字列の変数として "appSecret" という名称で保存する
9. Azure Automation アカウントで、Runbook を作成し、ランタイム バージョン 5.1 の PowerShell 形式とする
10. 上記 Runbook に以下のスクリプトを張りつける
11. スクリプトの中の $tenantId、$appId、$jobId は環境に合わせて書き換えて、スクリプトを発行する
12. Runbook の動作を検証し、スクリプトにより CSV のデータ取り込まれたことを確認する
13. HR から取り込む CSV ファイルを Azure Storage へのアップロード頻度に応じて、Runbook のスケジュール実行を設定する

### Azure Automation に登録するスクリプト本体
````
$StorageConnection = Get-AutomationVariable -Name 'storageKey'
$strctx = New-AzureStorageContext -ConnectionString $StorageConnection
$appSecret = Get-AutomationVariable -Name 'appSecret'
$tenantId="xxxxx" #Azure AD のテナント ID
$appId="xxxxx" #Azure AD で IRM 用に登録したアプリケーションの ID
$jobId="xxxx" #IRM の HR コネクタ設定で指定された ID

$path=Get-Location
Get-AzureStorageFileContent -ShareName "hrdata" -Path "HR_Resignation.txt" -Context $strctx
Get-AzureStorageFileContent -ShareName "hrdata" -Path "HRConnector.ps1" -Context $strctx
.\HRConnector.ps1 -tenantId $tenantId -appId $appId `
-appSecret $appSecret -jobId $jobId -filePath "$path\HR_Resignation.txt"
````

## HR Connector で取り込む CSV ファイルを元に、メールが有効なセキュリティ グループを作成し、グループメンバーシップをメンテナンスする
2 つ目のサンプルは、"IRMTargetGroup" という名前のメールが有効なセキュリティ グループを作成し、HR からの CSV ファイルに記載されているユーザーを、"IRMTargetGroup" に登録するものです。HR からの CSV ファイル側で、ユーザーの増減があれば、それに合わせて、"IRMTargetGroup" のメンバーも変更します。この　"IRMTargetGroup" のグループを作成しておくことで、IRM のポリシーの範囲を限定することができます。同様の手法で、IRM の用途に限らず、Azure Storage 上の CSV ファイルを元に、特定のグループをメンテナンスすることもできます。

### 事前準備
#### 先の手順で実施していない場合
1. Azure 上に Storage アカウントを作成し、そこにファイル シェア "hrdata" を作成する
2. 上記ファイル シェアに、"HR_Resignation.txt"というファイル名で、 IRM HR Connector で取り込む CSV ファイルを、BOM 付き UTF-8 の形式でアップロードしておく
3. Azure Automation アカウントを作成する
4. Azure Automation アカウントで、1.のファイル シェアへの接続キーを暗号化された文字列の変数として "storageKey" という名称で保存する
#### 新規手順
5. Azure Automation アカウントで、"ExchangeOnlineManagement" のモジュールをギャラリーから取り込む
  (ランタイム バージョンは、5.1)
6. Azure Automation アカウントで、Exchange Online 接続用に "Office 365" という名称で、Exchange 管理者の権限を持つアカウントの ID/パスワードを登録する
7. Azure Automation アカウントで、Runbook を作成し、ランタイム バージョン 5.1 の PowerShell 形式とする
8. 上記 Runbook に以下のスクリプトを張りつけて発行する
9. Runbook の動作を検証し、"IRMTargetGroup" というメールが有効なセキュリティ グループが作成され、CSV に記載されたメンバーが登録されていることを確認する
10. HR から取り込む CSV ファイルを Azure Storage へのアップロード頻度に応じて、Runbook のスケジュール実行を設定する

### Azure Automation に登録するスクリプト本体
````
# Get the connection string for the storage share
$StorageConnection = Get-AutomationVariable -Name 'storageKey'
$strctx = New-AzureStorageContext -ConnectionString $StorageConnection

# Get the credential for Azure AD connection
$Credential = Get-AutomationPSCredential -Name "Office 365"

# Get the HR CSV file
$path=Get-Location
Get-AzureStorageFileContent -ShareName "hrdata" -Path "HR_Resignation.txt" -Context $strctx
$csv=Import-Csv -Path "$path\HR_Resignation.txt"

# Create a list of users in HR CSV
$TargetUsers=@()
foreach($line in $csv){
    $TargetUsers+=$line.EmailAddress.ToLower()
}

# Show current users in CSV
"Current users in CSV ("+$TargetUsers.count+" users)"
$TargetUsers

# Connect Azure AD
Connect-ExchangeOnline -Credential $Credential

# Get "IRMTargetGroup", if it doesn't exist, create it
$IRMTargetGroup="IRMTargetGroup"
$g=Get-DistributionGroup -Identity $IRMTargetGroup -ErrorAction Ignore
$members=@()
if($g -eq $null){
    $g=New-DistributionGroup -Name $IRMTargetGroup -Type "Security"
    "IRMTargetGroup is newly created."
}
else{
    "IRMTargetGroup is found."
    
    #Get current users in "IRMTargetGroup"
    $mem=Get-DistributionGroupMember -Identity $IRMTargetGroup -ResultSize 2000
    
    #Create a list of users in "IRMTargetGroup"
    foreach($m in $mem){
        if($m.PrimarySmtpAddress)
            {$members+=$m.PrimarySmtpAddress.ToLower()}
        else{$members+=($m.Identity+"@"+$m.OrganizationalUnitRoot).ToLower()}
    }
    
    #Show current users in IRMTargetGroup
    "Current users in IRMTargetGroup ("+$members.count+" users)"
    $members
}

# Remover users from "IRMTargetGroup" if they are not in HR CSV
foreach($m in $members){
    if(!$TargetUsers.Contains($m)){
        Remove-DistributionGroupMember -Identity $IRMTargetGroup -Member $m -Confirm:$false
        "$m is removed."
    }
}

# Add users to "IRMTargetGroup" if they are not in the group but are in HR CSV
foreach($u in $TargetUsers){
    if(!$members.Contains($u)){
        Add-DistributionGroupMember -Identity $IRMTargetGroup -Member $u
        "$u is added."
        }
}
Disconnect-ExchangeOnline -Confirm:$false
"Done"
````
