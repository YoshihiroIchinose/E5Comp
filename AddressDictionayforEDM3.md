# キーワード辞書に合わせて EDM で取り込む住所をトリミングする
EDM の主要素のマッチングにおいては該当の文字列をハッシュ化して比較するため部分一致は行われず、
前段となる SIT で検知された文字列と、EDM でハッシュ化されて取り込まれた文字列とで、厳密に範囲が一致している必要があります。   
   
住所を EDM の主要素として検知したい場合、丁番号の表記ゆれやマンション名の有無などの影響を排除するため、
住所においては郵便番号で識別される都道府県市区町村町域名までの範囲で一致をみるというのが実装方法の一つとなります。
(ただし、同じ都道府県市区町村町域名の値を持つ EDM の行がある場合には、さらに住所を細分化して重複を減らすことが必要となります。)    
   
このスクリプトでは、EDM に取り込みたいデータを下処理するためのもので、[こちらのページ](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AddressDictionayforEDM2.md)で生成した住所のキーワード辞書を元に、
住所のデータを、キーワード辞書に含まれる都道府県市区町村町域名までにトリミングするものです。
また本スクリプトでは、合わせて[こちらのページ](https://github.com/YoshihiroIchinose/E5Comp/blob/main/EDM_Preprocess.md)で解説している取り込むデータの標準化も行っています。

## 変換例　住所データをキーワード辞書に登録されている町域名までの住所に変換する
#### EDM の元の住所データ    
神奈川県横浜市神奈川区片倉1-5-4

#### キーワード辞書の内容   
～   
神奈川県横浜市神奈川区大口通   
神奈川県横浜市神奈川区大口仲町   
神奈川県横浜市神奈川区大野町   
神奈川県横浜市神奈川区片倉   
神奈川県横浜市神奈川区神奈川  
神奈川県横浜市神奈川区神奈川本町   
～   

### 変換後の住所データ 
神奈川県横浜市神奈川区片倉   

## PowerShell スクリプト
```
# EDM で取り込みたい顧客データの CSV ファイル (住所情報は、Address 列とする)
$sourceFile="C:\Data\EDMCustomers\CustomeTestData.csv"
# 変換後の CSV ファイルの出力先
$targetFile = "C:\Data\EDMCustomers\CustomeTestData_Normalized.csv"
# 住所のキーワード辞書
$dictionaryFile = "C:\Data\JPAddressDicwithVariations.txt"

Function GetMaximumMatched($text){
	$prev=""
	Foreach($d in $dic){
		Switch($text.CompareTo($d)){
			1 {If($text.StartsWith($d)){$prev=$d}}
			0 {return $text}
			-1{return $prev}
		}
	}
	return $prev
}

$dic=Get-Content $dictionaryFile
# 最長一致を見つけるために、取り込んだ辞書情報をソートしておくことが重要
$dic = $dic | Sort-Object | Where-Object {$_.StartsWith("_") -eq $false}
$csv=Import-Csv -Path $sourceFile

Foreach($c in $csv){
  # 取り込む EDM データの全角英数記号などを半角英数に標準化しておく
	Foreach($att in $c|get-member|Where-Object{$_.MemberType -eq "NoteProperty"}){
		$c.($att.Name)=$c.($att.Name).Normalize([System.Text.NormalizationForm]::FormKC)
	}
　# Address 列の値をトリミング
	$c.Address=GetMaximumMatched $c.Address
}
$csv | Export-Csv -Path $targetFile -NoTypeInformation -Encoding unicode
```
