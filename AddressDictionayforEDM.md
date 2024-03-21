# EDM での日本語住所検知用にキーワード辞書を作成する
このページでは、EDM での日本語住所検知用にキーワード辞書を作成する手法について説明します。

## EDM での日本語住所検知でキーワード辞書が必要となる理由
EDM の主要素の検知では、前段となる SIT を定義し、その SIT で EDM のマッチング候補となる文字列を抽出することが必須となっています。
日本語の住所そのものを検知する場合、SIT で正規表現を利用することや、Named Entity を利用することで検知自体は簡単に行えます。
ただ、EDM で利用する場合、ハッシュ化して取り込んだデータと比較するため、SIT で検出する文字列と、EDM で取り込む住所のハッシュとで、
厳密に範囲が一致している必要があります。正規表現や、Named Entity では、住所のどこまでの範囲をマッチング候補と抽出するか制御することが難しく、
SIT で検知する住所の範囲で揺らぎがあると、EDM で取り込んだ住所のハッシュとで、マッチングしないことが起こります。

キーワード辞書であれば、基本的にはキーワードで登録した文字列がそのまま検出されるため、SIT として検出する範囲を厳密に定義することができます。
EDM の住所の検知においては、このように SIT としてキーワード辞書を用いて、
またこのキーワード辞書で定義する範囲でEDM 側の住所データも用意することで、範囲を合わせたマッチング行えます。

## どの範囲までをキーワード辞書に登録するか
一般に町番号、番地番号、号番号になると漢数字と英数字の表記ゆれ、‐で書くか、丁目と書くか、またマンション名が入る・入らないなど、ぶれる幅が大きくなります。
そのため、丁番号やマンション名が入らない、都道府県+市区町村+町名等までを比較対象とすることが、望ましいかと考えられます。
こういったキーワード辞書を作成する場合、実際に EDM で利用する顧客の住所データをトリミングして生成することがベストですが、
より汎用的に住所を検知しておく場合、このページで紹介する郵便番号で識別される住所をキーワード辞書に登録して利用することもできます。
ただし、EDM のデータ取り込みの際には、事前にキーワード辞書と合わせた範囲に住所データをトリミングした上で、ハッシュ化する必要があります。

## 元となる日本郵便の住所情報
日本郵便では、郵便番号で識別される住所一覧を CSV 形式で提供しており、これを元に、PowerShell スクリプトを用いて、
日本の住所のキーワード辞書を再現性のある形で生成します。
元となるデータは、1 レコード 1 行、UTF-8 形式のものをダウンロードして利用します。
[郵便番号データダウンロード](https://www.post.japanpost.jp/zipcode/download.html)

## キーワード辞書のための処理
EDM の前段となる SIT 用のキーワード辞書のために、日本郵便の郵便番号の住所データを以下の観点で加工します。
1. CSV データの 7-9 列目に当たる、都道府県、市区町村名、町域名を連結してキーワード辞書に登録する住所とする
1. 全角英数記号は、半角英数記号に変換する
1. 同じ住所の範囲でも、() がついて、西側と東側などで、郵便番号が異なるケースがあるが、() 以降は無視する
1. 3 の処理の結果、重複する住所となる場合には、キーワード辞書に登録しない
1. 同じ郵便番号を共有する町域名が "、" により併記されている場合には、都道府県、市区町村を保管した上で、別行としてキーワード辞書に登録する
1. 数字の後に号、線、番、丁、区、条、の通り、入会、牧場などが続くケースや、第の後に数字が続くケースでは、数字を漢数字に変換したパターンもキーワード辞書に登録する
1. 岩手県の地割では、～で複数の地割を一つの郵便番号となっているケースなどもあるが、数字・地割部分は除いたものを重複なくキーワード辞書に登録する
1. 町域名に"以下に掲載がない場合"とある記載は無視して、都道府県、市区町村名までとする
1. 上の処理や、そもそもの住所データにより、部分住所がデータ上包含関係がある行の存在を許容する

## 生成されたキーワード辞書
日本郵便の 2024 年 2 月 29 日のデータを用いて生成されたキーワード辞書のデータが[こちら](https://github.com/YoshihiroIchinose/E5Comp/blob/main/WB/JPAddressDic.txt)となります。ただし、このデータだけでは、後に続く文字列に依存したワードブレークのブレには対応しないため、[こちら](https://github.com/YoshihiroIchinose/E5Comp/edit/main/AddressDictionayforEDM2.md)のページも併せて参照して下さい。

## スクリプト本体
```
#$a[55]="五十五"となるような配列を事前に作成しておく
$a=("","一","二","三","四","五","六","七","八","九")
$b=("十","十一","十二","十三","十四","十五","十六","十七","十八","十九")
$a+=$b
for($i=2;$i -le 9; $i++){
	Foreach($j in $b)
	{
		$a+=$a[$i]+$j
	}
}
#数字と関連する前後の文字
$suffix="([号線番丁区条]|の通り|入会|牧場)"
$prefix="(第)"

#地割がある場合には、数字地割の部分と第の部分を除く
Function IgnoreZiwari($original){
	If($original -match '(\d{1,2})'+"地割"){
		$temp=$original.Substring(0,$original.IndexOf($Matches[0]))
		If($temp[$temp.length-1] -eq "第"){
			Return $temp.Substring(0,$temp.length-1)
		}
		Return $temp 
	}
	Return $original
}

#数字が含まれる部分住所では、漢数字に置き換えたパターンも用意する
#1 桁の数位が 2 回出てくるパターンも考慮
Function variations($original){
	$normalized=""
	If($original -match '(\d{2})'+$suffix){
		$normalized=$original.replace($Matches[0], $a[$Matches[1]]+$Matches[2])
	}else{ 
		if($original -match '(\d)'+$suffix){
		$normalized=$original.replace($Matches[0], $a[$Matches[1]]+$Matches[2])
	  	if($normalized -match '(\d)'+$suffix){
			$normalized=$normalized.replace($Matches[0], $a[$Matches[1]]+$Matches[2])
		  }
	  }
	}
	If($original -match $prefix+'(\d)$'){
  	$normalized=$original.replace($Matches[0],$Matches[1]+$a[$Matches[2]])
	}
	If($normalized -eq ""){
		Return $original
	}
	Else{
		Return $original+"`r`n"+$normalized
	}
}

#"、"で町域名が併記される場合を分割する
Function SpritParallel($original){
	$parts=$original.split("、")
	If($parts.count -eq 1){return $original}	
	If($parts[1].length -ge 2){
			$parts[1]=$parts[0].Substring(0,$parts[0].length-2)+$parts[1]
	}
	If($parts[1].length -eq 1){
			$parts[1]=$parts[0].Substring(0,$parts[0].length-1)+$parts[1]
	}
	If($parts.count -eq 3){
			$parts[2]=$parts[0].Substring(0,$parts[0].length-2)+$parts[2]
	}
	Return $parts
}

#日本郵便からダウンロードした 1 レコード 1 行、UTF-8 形式のデータ
$sourceFile="C:\WB\utf_ken_all.csv"
$source = New-Object System.IO.StreamReader($sourceFile, [System.Text.Encoding]::GetEncoding("utf-8"))

#出力先のファイル
$targetFile = "C:\WB\JPAddressDic.txt"
$target = New-Object System.IO.StreamWriter($targetFile, $false, [System.Text.Encoding]::GetEncoding("UTF-16LE"))

$prev=""
while (($line = $source.ReadLine()) -ne $null){
	$data=$line.Split(",")
	$address=$data[6]+$data[7]+$data[8]
	$address=$address.Normalize([System.Text.NormalizationForm]::FormKC)
	$address=$address.Replace("以下に掲載がない場合","")
	$address=$address.Replace("`"","")
	
	If($data[8].IndexOf("の次に") -ne -1){
		If($data[8].StartsWith("琴平町の次に")){continue}
		$address=$data[6]+$data[7]
	}
	$s=$address.LastIndexOf("(")
	$e=$address.LastIndexOf(")")
	If($e -gt $s -and $s -gt 0){
		$address=$address.Substring(0,$s)
	}
	$address=IgnoreZiwari($address)
	If($address.Equals($prev)){
		continue
	}
	$prev=$address
	Foreach($p in SpritParallel $address)
	{
		$nomalized=variations $p
		$target.WriteLine($nomalized)
	}
}
$source.Close()
$target.Close()
```
