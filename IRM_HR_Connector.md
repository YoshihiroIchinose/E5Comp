# Insider Risk Management の HR Connector に関連したスクリプト
## HR Connector で取り込む CSV ファイルを元に、メールが有効なセキュリティ グループを作成し、グループメンバーシップをメンテナンスする
### 事前準備

### Azure Automation に登録するスクリプト本体
````
$StorageConnection = Get-AutomationVariable -Name 'storageKey'
$strctx = New-AzureStorageContext -ConnectionString $StorageConnection
$path=Get-Location
Get-AzureStorageFileContent -ShareName "hrdata" -Path "HR_Resignation.txt" -Context $strctx
Get-AzureStorageFileContent -ShareName "hrdata" -Path "HRConnector.ps1" -Context $strctx
.\HRConnector.ps1 -tenantId "394a6b35-2c31-4e9a-b74e-dbf4ceeb904f" -appId "xxxx"  -appSecret "xxxx"  -jobId "xxxx" -filePath "$path\HR_Resignation.txt"
````

## HR Connector で取り込む CSV ファイルを元に、メールが有効なセキュリティ グループを作成し、グループメンバーシップをメンテナンスする


### 事前準備
1. Azure 上に Storage アカウントを作成し、そこにファイル シェアを作成する
2. 上記ファイル シェアに、"HR_Resignation.txt"というファイル名で、 IRM HR Connector で取り込む CSV ファイルを、BOM 付き UTF-8 の形式でアップロードしておく
3. Azure Automation アカウントを作成する
4. Azure Automation アカウントで、"ExchangeOnlineManagement" のモジュールをギャラリーから取り込む
  (ランタイム バージョンは、5.1)
5. Azure Automation アカウントで、1.のファイル シェアへの接続キーを暗号化された文字列の変数として "storageKey" という名称で保存する
6. Azure Automation アカウントで、Exchange Online 接続用に "Office 365" という名称で、管理者アカウントの ID/パスワードを登録する
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
#Create a list of users in HR CSV
$TargetUsers=@()
foreach($line in $csv){
    $TargetUsers+=$line.EmailAddress.ToLower()
}
#Show current users in CSV
"Current users in CSV ("+$TargetUsers.count+" users)"
$TargetUsers
#Connect Azure AD
Connect-ExchangeOnline -Credential $Credential
#Get "IRMTargetGroup", if it doesn't exist, create it
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
#Remover users from "IRMTargetGroup" if they are not in HR CSV
foreach($m in $members){
    if(!$TargetUsers.Contains($m)){
        Remove-DistributionGroupMember -Identity $IRMTargetGroup -Member $m -Confirm:$false
        "$m is removed."
    }
}
#Add users to "IRMTargetGroup" if they are not in the group but are in HR CSV
foreach($u in $TargetUsers){
    if(!$members.Contains($u)){
        Add-DistributionGroupMember -Identity $IRMTargetGroup -Member $u
        "$u is added."
        }
}
Disconnect-ExchangeOnline -Confirm:$false
"Done"
````
