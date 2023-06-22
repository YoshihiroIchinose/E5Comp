# SharePoint Online および OneDrive for Business の各サイトに設定されたゲスト ユーザーをリストする (SPO Shell)
本スクリプトは、Azure Automation 上に作成したスクリプトから、SharePoint Online Management Shell を利用して、
SharePoint Online および　OneDrive for Business の各サイトでのゲスト ユーザー一覧を取得するものです。
なお取得した外部ユーザー一覧の情報は、CSV ファイルとして、特定の SharePoint Online サイトのドキュメント ライブラリに格納します。
なおクエリの仕組み上、Teams での外部招待を含みSharePoint グループに外部ユーザーが登録されている場合や、
直接外部ユーザーに権限が付与されている場合はゲスト ユーザーを検知できますが、セキュリティ グループを介した間接的な外部ユーザーへの権限付与は取得されません。
外部ユーザーが参加しているグループは、Azure AD 上の外部ユーザーの memberof 属性で確認できます。

## 準備
1. Azure 環境にて Azure Automation アカウントを作成
2. Azure Automation アカウントにて、"モジュール" -> "ギャラリーを参照"から、以下のモジュールを追加する。   
(ランタイム バージョンは 5.1)   
  SharePointOnline.CSOM    
  Microsoft.Online.SharePoint.PowerShell　　　　
3. Azure Automation アカウントの"資格情報"->"資格情報の追加"で、SharePoint Online の管理者権限があり、指定の SharePoint Online サイトに投稿権限があるアカウントの ID とパスワードを "Office 365" という名称で登録しておく。
5. Azure Automation アカウントの"Runbook"->"Runbook の作成"で PowerShell、ランタイム バージョンの 5.1 の Runbook を作成する
6. 作成した Runbook に以下のスクリプトをコピー & ペーストする
7. 適宜スクリプト内の SharePoint Site の URL および、ファイル保存先の相対 URL (FQDN を除いたもの) 保存し、公開する
8. 作成した Runbook を"開始"し、動作を確認する

## Azure Automation 実行時の出力
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/SPOExt1.png">   
## 出力される CSV ファイル
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/SPOExt2.png">   

## スクリプト本体
```
$siteUrl="https://xxxx.sharepoint.com/sites/CustomNotification/"
$targeturl ="/sites/CustomNotification/Shared Documents/ExternalUsersFromSPO.csv"
$adminSiteUrl="https://xxxx-admin.sharepoint.com"
$OutputFile=[System.Environment]::GetFolderPath("Desktop")+"\ExternalUsersFromSPO.csv"
$Credential = Get-AutomationPSCredential -Name "Office 365"

Function AddMember{
    Param($a,$b,$c)
    add-member -InputObject $a -NotePropertyName $b  -NotePropertyValue $c
}

Connect-SPOService -Url $adminSiteUrl -Credential $Credential

$Sites=Get-SPOSite -Limit ALL  -IncludePersonalSite $true
$externalUsers=@()
Foreach($s in $sites){
	$users=Get-SPOExternalUser -Position 0 -PageSize 50 -SiteUrl $s.Url
	"$($users.count) external users in  $($s.Url)"
	If($users.count -eq 50){"Obtained external users might be capped."}
	Foreach($u in $users){
	         $line = New-Object -TypeName PSObject
		 AddMember $line "Site" $s.Url
		 AddMember $line "DisplayName" $u.DisplayName
      　　　　　  AddMember $line "UniqueId" $u.UniqueId
		 AddMember $line "Email" $u.Email
		 AddMember $line "LoginName" $u.LoginName
		$externalUsers+=$line
	}
}

"Total external users found:"+$userList.count
$externalUsers|Export-Csv -Path $OutputFile -NoTypeInformation -Encoding UTF8

#Uploading a csv file to SPO
Load-SPOnlineCSOMAssemblies
$ctx = New-Object Microsoft.SharePoint.Client.ClientContext($siteUrl)
$username =$Credential.UserName
$password = $Credential.Password
$cre=$null
$count=0
while($cre -eq $null -and $count -lt 10){
    $count++
    try{$cre = New-Object Microsoft.SharePoint.Client.SharePointOnlineCredentials($username,$password)}
    catch{
            "Authentication Error!"
            $_.Exception.Message
            Start-Sleep -s 5
    }
}
$ctx.Credentials=$cre

$fs = new-object System.IO.FileStream($OutputFile ,[System.IO.FileMode]::Open,[System.IO.FileAccess]::Read)
[Microsoft.SharePoint.Client.File]::SaveBinaryDirect($ctx,$targeturl , $fs, $true)
$fs.Close()
$ctx.Dispose()
"Upload completed."
```
