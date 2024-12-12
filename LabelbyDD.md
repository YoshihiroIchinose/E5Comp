# 秘密度ラベルの操作をデスクトップ上のドラッグ アンド ドロップの操作で行う
本スクリプトは、Purview Information Protection Client に含まれる
ラベル付けおよびラベル削除のPowerShell コマンドレットを用いて、
ドラッグ アンド ドロップの操作で、複数のファイルに対して秘密度ラベル付けや、ラベルの削除を行うサンプルです。
## 概要
PowerShell のスクリプトそのものは、ドラッグ アンド ドロップの操作を受け付けないので、
PowerShell のスクリプトを呼び出すショートカットを作成しておき、ドラッグ アンド ドロップの操作は、
そのショートカット ファイルに対して行うようにします。
なおラベルを付けるスクリプトでは、付与したいラベルの GUID を指定する必要があり、
ラベルを削除するスクリプトでは、管理者によるラベルの設定によっては、理由の文言を指定する必要があります。

## 事前準備
1. [こちら](https://aka.ms/aipclient)から Purview Information Protection Client をインストールする
2. PowerShell を管理権限で立ち上げて事前に一度以下を実行する
```
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
```
3. PowerShell で Get-FileStaus に自動で設定したい秘密度ラベルを付与したファイルのパスを指定して、秘密度ラベルの GUID を把握しておく
```
Get-FileStatus "ファイルのパス"
```
上記を実行して返ってきた、SubLabelId が対象の秘密度ラベルの GUID となる
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/LabelStatus.png">
## 一括の秘密度ラベル付け設定手順
1. メモ帳で以下のスクリプトを張り付け、事前準備 3 で取得した秘密度ラベルの GUID を指定するよう変更し、Labeling.ps1 として保存する
```
$label="ラベルの GUID "

$Args | foreach{
    $item = Get-Item -LiteralPath $_
    $item.FullName
    $status=get-filestatus $item
    if($status.SubLabelId -ne $label){
	$status=Set-FileLabel -path $item -Labelid $label
	"The operation of labeling was " +$status.Status
    }else{
     "The label has already been set."
    }
    
}
```
2. 1 のスクリプトを呼び出すショートカットを作成する
ショートカットのリンク先として以下指定行い、スクリプトの場所を指定する。
```
powershell.exe -NoProfile -ExecutionPolicy RemoteSigned -noexit -File "C:\Users\MeganBowen\Desktop\Labeling.ps1"
```
3. 上記ショートカットにラベル付けを行いたいファイルをドラッグ アンド ドロップすれば一括でラベル付けが可能

## 一括の秘密度ラベル削除の設定手順
1. メモ帳で以下のスクリプトを張り付け、$Justificationの部分を適宜書き換え、RemovingLabel.ps1 として保存する
```
$Justification="For sharing with external customers"

$Args | foreach{
    $item = Get-Item -LiteralPath $_
    $item.FullName
    $status=get-filestatus $item
    if($status.MainLabelName -eq $null){
	"No label"
         exit
	}
    "Current label:" + $status.MainLabelName +" / "+ $status.SubLabelName
    if($status.SubLabelId -ne $null){
	$status=Remove-FileLabel -path $item -JustificationMessage $Justification
	"The operation of label removal was " + $status.Status
    }
}
```
2. 1 のスクリプトを呼び出すショートカットを作成する
ショートカットのリンク先として以下指定行い、スクリプトの場所を指定する
```
powershell.exe -NoProfile -ExecutionPolicy RemoteSigned -noexit -File RemovingLabel.ps1
```
3. 上記ショートカットにラベル付けを行いたいファイルをドラッグ アンド ドロップすれば一括でラベル削除が可能

## その他考慮事項
本スクリプトの範囲では、ユーザーの権限を用いているため、ユーザーがフルコントロール権限を有していない場合、既に保護されているファイルのラベル変更は行えません。
管理者などがファイルの復旧に利用する場合には、[SuperUser](https://learn.microsoft.com/ja-jp/azure/information-protection/configure-super-users) の設定をして、
そのユーザー アカウントを利用して操作する必要があります。   
PowerShell のスクリプトを展開して他のデバイスなどでも利用できるようにするためには、各ユーザー側で、スクリプトを右クリックしてプロパティを表示し、許可するか、
スクリプトを信頼された認証局の証明書で[署名](https://learn.microsoft.com/ja-jp/powershell/module/microsoft.powershell.core/about/about_signing?view=powershell-7.4)して、改ざんを検知できるようにしておく必要があります。
