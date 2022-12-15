# Communication Complinance のポリシーで合致した内容を一括ダウンロードする
## 概要
Communication Complinace では、ポリシーごとにメールボックスが作成されているため、このメールボックスを対象とした、
コンテンツ検索を実施することで、ポリシーに合致したメッセージを一括で .pst 形式でダウンロードすることが可能です。
ここでは、Communication Compliance の特定のポリシーに合致したメッセージを抽出する操作を自動化するサンプル スクリプトをご紹介します。

## 動作について
コンテンツ検索では、-AllowNotFoundExchangeLocationsEnabled $true のパラメーターを使用することで、
こういったシステムで持っている非表示のメールボックスも対象にした検索が可能となっています。
ここでのコンテンツ検索では、特定の日付以降に送信されたメッセージを対象に抜き出していますが、
適宜検索条件を書き換えることで、抽出する範囲を変更できます。    
コンテンツ検索を行うためには、適宜権限の設定も必要となります。<br>
[参考: コンプライアンス ポータルで電子情報開示のアクセス許可を割り当てる](https://learn.microsoft.com/ja-jp/microsoft-365/compliance/assign-ediscovery-permissions?view=o365-worldwide)

## 自動化の範囲について
コンテンツ検索の仕組みでは、検索の定義、検索の開始、検索完了までの待機、エクスポートの開始、エクスポート完了までの待機、エクスポート結果のダウンロードといった手順がありますが、
最後のエクスポート結果のダウンロードは、PowerShell のみでは完結できず、ClickOnce の .NET アプリを通じた対話的な対応が必要で、このスクリプトでは、ClickOnece の .NET アプリの
起動までを自動化しています。ClickOnece のアプリ起動後は、クリップボードにコピーしてあるエクスポート キーを貼り付けて、ダウンロード場所を指定して、ダウンロードを行います。
ダウンロードした .pst ファイルは、Outlook のファイル メニュー -> アカウント設定 -> データファイルにある追加で、Outlook に読み込むことができます。<br>
[参考: コンテンツ検索のレポートをエクスポートする](https://learn.microsoft.com/ja-jp/microsoft-365/compliance/export-a-content-search-report?view=o365-worldwide)

## スクリプト サンプル
````
#CCのポリシー名を指定
$CCPolicyName="CCPolicy"

#検索条件
$Query="sent>=2022-11-01T00:00:00+09:00"

$SearchName=$CCPolicyName+" at "+(Get-Date -Format "yyyy-MM-dd HHmm")
Connect-ExchangeOnline
$p=Get-SupervisoryReviewPolicyV2 -Identity $CCPolicyName

#CCのポリシーに紐づくメールボックスに対して、特定の日付以降の送信日時のコンテンツ検索を定義する
Connect-IPPSSession
New-ComplianceSearch $SearchName -ExchangeLocation $p.ReviewMailbox -AllowNotFoundExchangeLocationsEnabled $true -ContentMatchQuery $Query

#検索をキックして完了を待つ
Start-ComplianceSearch -Identity $SearchName

Do{
Start-Sleep -s 30
$s=Get-ComplianceSearch -Identity $SearchName
(Get-Date -Format "HHmm") +" "+ $s.Status
}while($s.Status -ne "Completed")


#検索結果のエクスポートをキックして待つ
New-ComplianceSearchAction -SearchName  $SearchName -Export -ExchangeArchiveFormat SinglePst -Format FxStream

Do{
Start-Sleep -s 30
$a=Get-ComplianceSearchAction -identity ( $SearchName  + '_Export')
(Get-Date -Format "HHmm") +" "+ $a.Status
}while($a.Status -ne "Completed")

#Edge 経由での ClickOnce のアプリ起動をする
$a=Get-ComplianceSearchAction -identity ( $SearchName  + '_Export') -includeCredential
$temp=$a.Results.Substring(0,$a.Results.IndexOf(";"))
$url=[System.Uri]::EscapeDataString($temp.Substring($temp.IndexOf("https://")))

$token=$a.Results.Substring($a.Results.IndexOf("SAS token: ")).split(";")[0].split(" ")[2]
$ActionName=$a.Name.Replace(" ","+")

#エクスポート キーをクリップボードにコピーしておく
$token | clip

#Edge 側で管理者アカウントで認証が通っていれば、ClickOnce のコンテンツ ダウンロード アプリが立ち上がるので、
#クリップボードから、エクスポート キーをペーストとして、.pst ファイルをローカルにダウンロードする
start microsoft-edge:"https://complianceclientsdf.blob.core.windows.net/v16/Microsoft.Office.Client.Discovery.UnifiedExportTool.application?source=$url&name=$ActionName&trace=1&lite=1&customizepst=1"

````
