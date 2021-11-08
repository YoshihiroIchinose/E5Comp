# カスタムの機密情報定義で正規表現のエラーが出た場合
Office 365 の Data Classfication Service では、現在 C++ の Boost.Regex 5.1.3 のエンジンが利用されています。  
[参考： カスタムの機密情報の種類を使用する前に - Microsoft 365 Compliance | Microsoft Docs](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/create-a-custom-sensitive-information-type?view=o365-worldwide#before-you-begin)  
カスタムの機密情報定義の正規表現についてですが、手動で登録したカスタムの機密情報定義は内部的にすべて共通のルール パッケージにまとめられおり、他のどれかの正規表現に仕様変更などでエラーが発生すると、エラーがないカスタムの機密情報定義の編集もすべてエラーとなってしまいます。 
 
### 正規表現がエラーとなってしまったため、新規の機密情報の作成や、既存の機密情報の定義の更新ができなくなった場合に表示されるエラー
このエラーでは、 "クライアント エラー 指定された分類ルール コレクションに、無効な正規表現プrオセッサが含まれています。無効な正規表現プロセッサの識別子は "" です。"   と表示されています。
![SIT Error](https://github.com/YoshihiroIchinose/E5Comp/blob/main/Error_SIT.png)

上記のようなエラーが出た場合、エラーとなっている正規表現が一か所であれば、該当の機密情報の定義を編集し、エラーがない状態に戻せば OK です。ただ、同種の正規表現を複数の機密情報の定義で使っていた場合、Compliance Cetner の Web UI 上から同時に更新できるのは一つであるため、UI 上からこれらのエラーに対処することはできません。そういった場合、以下のようなコマンドで設定を XML で抜き出し、一括で編集後再アップロードすれば OK です。

## 機密情報定義の一括のエクスポート
```
#接続に利用するID / パスワード
$Id="admin@xxxx.onmicrosoft.com"
$Password = "xxxxx"
$outfile ="C:\Comp\SIT.xml"

#Credentialの生成
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Connect-IPPSSession -UserPrincipalName $id -Credential $Credential
# UI から定義した機密情報は、"Microsoft.SCCManaged.CustomRulePack" という名前で、1 つの RulePack としてまとめられている。
$r=Get-DlpSensitiveInformationTypeRulePackage |?{$_.LocalizedName -eq "Microsoft.SCCManaged.CustomRulePack"}
$r.ClassificationRuleCollectionXml |out-file $outfile
```
## XML の編集
抽出したカスタムの機密情報定義が、"C:\Comp\SIT.xml" に保存されているので、こちらの XML ファイルを編集し、エラーとなっている機密情報の定義を適宜削除・修正し上書き保存します。

## 編集した XML を再度アップロード
```
Set-DlpSensitiveInformationTypeRulePackage -FileData (Get-Content -Path $outfile -Encoding Byte)
```

## 参考 2021/11/04 に確認した正規表現のエラー
電話番号の定義の正規表現で利用していた以下のアサーションがこれまで利用出来ていたもの今回新規エラーとなった。  
### 見つかった電話番号の前に半角英数字、各種ハイフンおよび、半角ハイフン+半角スペースがないことを確認するエラーとなる先頭のアサーション
```
(?<!\w|[\－ー―-]|- )
```
### 見つかった電話番号の後ろに半角英数字、各種ハイフンおよび、半角スペース＋ハイフンがないことを確認するエラーとなる最後尾のアサーション
```
(?!\w|[\－ー―-]| -)
```
上記がなぜエラーになるかについて調べた所、Boost.Regex 5.1.3 の [Perl 向けの Reference](https://www.boost.org/doc/libs/1_68_0/libs/regex/doc/html/boost_regex/syntax/perl_syntax.html) に Lookahead および Lookbehind は固定長でなければいけないとの記載があり、確かに上のアサーションでは、1 文字の場合と、半角スペースと半角ハイフンの場合とで、1 文字ないしは、2 文字のパターンを併記していたためエラーとなっていることが判明。Office 365 の DLP においては、全角数字と半角ハイフンとで構成された電話番号は、全角数字が半角数字に変換されるだけではなく、全角と半角の間にスペースを入れる事前処理を行った上で、マッチングが行われるので、元データが全角数字と半角ハイフンを使った表記の場合、電話番号として正しくない長さの文字列が電話番号として過検知される可能性があるものの、その部分の除外は除くとして、修正するとすれば、半角スペース+半角ハイフンのケースを除いた以下のパターンへの修正が必要であった。
### 見つかった電話番号の前に半角英数字および各種ハイフンがないことを確認する修正後の先頭のアサーション
```
(?<!\w|[\－ー―-])
```
### 見つかった電話番号の後ろに半角英数字および各種ハイフンがないことを確認する修正後の最後尾のアサーション
```
(?!\w|[\－ー―-])
```

### 修正後の電話番号定義
日本の電話番号としての正規表現の例としては以下とした。こちらの[日本向け個人情報の定義](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SIT.md)も参照のこと。
```
(?<!\w|[\－ー―-])0(\d([\－ー―-]| - )\d{3}|\d{2}([\－ー―-]| - )\d{2}|\d{3}([\－ー―-]| - )\d|\d{4}([\－ー―-]| - )|[5789]0([\－ー―-]| - )\d{3})\d([\－ー―-]| - )\d{4}(?!\w|[\－ー―-])
```
