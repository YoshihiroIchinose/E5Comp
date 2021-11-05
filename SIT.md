# 日本向け個人情報の定義
Office 365 DLP 向けに日本での個人情報に該当するパターンを定義するものです。現時点で住所と電話番号を定義しています。用語は、カスタムの機密情報の定義として [XML](https://github.com/YoshihiroIchinose/E5Comp/blob/main/JPN_SIT.xml) ファイルで用意していますので、ダウンロードの上、PowerShell で Office 365 に取り込んで利用ください。他にも日本向けの Communication Compliance 用の用語集の定義はこちらの [ページ](https://github.com/YoshihiroIchinose/JPN-CC/blob/master/README.md) で紹介しています。

# 事前に一度実施
## PowerShell を管理権限で立ち上げて以下を実行
    Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
    Install-Module -Name ExchangeOnlineManagement
    
# カスタムの機密情報として XML を取り込み
## PowerShell より Exchange Online に接続
    Import-Module ExchangeOnlineManagement
    Connect-IPPSSession

## ローカルの XML ファイルをアップロード
    New-DlpSensitiveInformationTypeRulePackage -FileData (Get-Content -Path "C:\Users\imiki\Desktop\Work\Comp\JPN_SIT.xml" -Encoding Byte)

## 一度アップロード済みで新しいバージョンの定義に更新の場合
    Set-DlpSensitiveInformationTypeRulePackage -FileData (Get-Content -Path "C:\Users\imiki\Desktop\Work\Comp\JPN_SIT.xml" -Encoding Byte)
    
# DLP のポリシーを設定
取り込んだ以下のカスタムの個人情報定義を利用して、Office 365のDLP のポリシー等を設定下さい。それぞれ該当するものが見つかれば信頼度 80 でマッチします。  
SIT1.住所  
SIT2.電話番号  
SIT3.メールリンク  
SIT4.和暦  
# SIT1.住所
以下の正規表現で都道府県＋市区町村＋全角文字で構成される文字列をマッチングし日本での住所の表現を検出します。
## 以下のようなパターンを検出します。  
東京都港区港南  
北海道札幌市中央区  

## 正規表現
    (北海道|東京都|(大阪|京都)府|(神奈川|和歌山|鹿児島)県|[^\x00-\x7F]{2}県)\s*[^\x00-\x7F]{1,6}[市郡区町村]

# SIT2.電話番号
## 以下の電話番号のパターンを検出します。  
2 桁市外局番 0x-xxxx-xxxx  
3 桁市外局番 0xx-xxx-xxxx  
4 桁市外局番 0xxx-xx-xxxx  
5 桁市外局番 0xxxx-x-xxxx  
IP 電話 050-xxxx-xxxx  
携帯 070-xxxx-xxxx, 080-xxxx-xxxx, 090-xxxx-xxxx  
## 対象外のパターン
市外局番の() (0x)-xxxx-xxxx  
フリーダイヤル 0120-xxx-xxx  
国番号含む表現 +81-x-xxxx-xxxx  

## Office 365 での注意事項
### 1. 全角英数字が半角英数字に事前に変換される
Office 365 内のテキストのマッチングにおいては、全角英数字は半角英数字に変換された上で正規表現とのマッチングが行わます。  
### 2. 全角に挟まれた半角ハイフンの扱い
"全角-全角"のように全角に挟まれた半角ハイフンは前後にスペースが挿入され" - "に置換された上で、マッチングが行われる点に注意します。  

## 電話番号の正規表現
### 採用している電話番号の正規表現
    (?<!\w|[\－ー―-])0(\d([\－ー―-]| - )\d{3}|\d{2}([\－ー―-]| - )\d{2}|\d{3}([\－ー―-]| - )\d|\d{4}([\－ー―-]| - )|[5789]0([\－ー―-]| - )\d{3})\d([\－ー―-]| - )\d{4}(?!\w|[\－ー―-])

### 上記の解説  
#### \d
半角の数字 1 桁を示しています。  
#### \d{4}
半角の数字 4 桁を示しています。  
#### ([\－ー―-]| - )
全角もしくは半角のハイフンとハイフンに類似する全角文字、注意事項 3 に留意した変換後の" - "のいずれかを示しています。(注:Office 365の機密情報定義で全角ハイフンにもエスケープが必要になったため－については\－で表現するように変更)  
#### [5789]
半角の 5,7,8,9 のいずれか一文字を示しています。  
#### \w
[A-Za-z0-9] と同等で半角英数1文字を示しています。  
#### (?<!\w|[\－ー―-])
他の部分がマッチした後、前に半角の英数字やハイフンがないことを確認する正規表現のアサーションとなっており、電話番号の形式を部分的に含む、E03-0000-0000 のようなシリアル コードなどの類似する文字列を排除するようにしています。アサーションでは、固定長のパターンが求められるので、以前の定義を見直しました。  
#### (?!\w|[\－ー―-])
他の部分がマッチした後、後ろに半角の英数字やハイフンがないことを確認する正規表現のアサーションとなっており、電話番号の形式を部分的に含む、03-0000-0000-0000 のようなシリアル コードなどの類似する文字列を排除するようにしています。アサーションでは、固定長のパターンが求められるので、以前の定義を見直しました。

# ハイフンの種類
| 文字 | UTF-8 | Unicode Code Point | Description | 今回の対象かどうか |
|:---:|---:|---:|:---|:---:
| - | 2D | U+002D | Hyphen-Minus | x |
| ー | E383BC | U+30FC | Katakana-Hiragana Prolonged Sound Mark | x |
| ‐ | E28090 | U+2010 | Another Hyphen ||
| ‑ | E28091 | U+2011 | Non-Breaking Hyphen ||
| ‒ | E28092 | U+2012 | Figure Dash ||
| – | E28093 | U+2013 | En Dash ||
| — | E28094 | U+2014 | Em Dash ||
| ― | E28095 | U+2015 | Horizontal Bar | x |
| − | E28892 | U+2212 | Minus Sign ||
| ⁃ | E28183 | U+2043 | Hyphen Bullet ||
| ﹣ | EFB9A3 | U+FE63 | Small Hyphen-Minus ||
| ｰ | EFBDB0 | U+FF70 | Halfwidth Katakana-Hiragana Prolonged Sound Mark ||
| - | EFBC8D | U+FF0D | Fullwidth Hyphen-Minus | x |

# SIT3.メール リンク
メール アドレスとなる文字列ではなく、Office ファイルに記載され mailto のリンクになっているメールアドレスを検出します。  
## 正規表現
    mailto:[\w\-.!#$%&'*+\/=?^_`{|}~]+@[\w\-_]+\.[\w\-_]+

# SIT4.和暦
## 以下の和暦のパターンを検出します。
令和元年十二月十二日  
昭和 63 年 5 月 1 日  
西暦二〇〇〇年拾月吉日  
## 正規表現
    (明治|大正|昭和|平成|令和|西暦)\s*([\d{1,4}|[元一二三四五六七八九十壱弐参拾〇○零]{1,4})\s*年\s*([\d{1,2}|[一二三四五六七八九十壱弐参拾〇○]{1,2})\s*月\s*([\d{1,2}|[元一二三四五六七八九十壱弐参拾〇○吉]{1,2})\s*日  

# 参考情報
1. Web で正規表現のチェックができる[正規表現チェッカー](http://okumocchi.jp/php/re.php)  
1. 文字のコードを確認できる [Unidode文字ツール](https://www.marbacka.net/msearch/tool.php#chr2enc)  
