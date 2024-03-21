# キーワード辞書でワード マッチにおけるワードブレークのずれに対処する
キーワード辞書に[先のページ](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AddressDictionayforEDM.md)で作成したキーワード辞書を登録しても、本文中から検知しない住所が存在します。これは、住所において町域等の後に数字で丁番号が続くことが一般的に想定されますが、これらの丁番号が続くか続かないかによって、ワードブレークの位置がずれてしまうケースがあるためです。

## 丁番号有無によるワードブレークのずれ
例キーワード辞書に登録した文字列   
Ⓐ愛知県愛西市南河田町   
   
本文中の住所例   
Ⓑ愛知県愛西市南河田町1   
Ⓒ愛知県愛西市南河田町1-2-3   
   
これらは、それぞれ以下のようなワードブレークが実施されます。   
Ⓐ’愛知 県 愛 西 市 南 河田 町   
Ⓑ'愛知 県 愛 西 市 南河 田 町 1   
Ⓒ'愛知 県 愛 西 市 南河 田町 1-2-3   

上記のケースでは、Ⓐ'Ⓑ'Ⓒ'それぞれで、異なるワード ブレークがなされており、ワード マッチの動作となるキーワード辞書においては、
Ⓐの文字列をキーワード辞書登録しても、本文中のⒷやⒸからは、Ⓐのワードを検出できないという現象が発生します。    

カスタムのアプリによるワードブレーク動作の結果   
(ワード ブレークを検証するためにはアプリを作成する必要があり、またワードブレークの呼び出しは簡単ではない。)   
<img width="457" alt="image" src="https://github.com/YoshihiroIchinose/E5Comp/assets/66407692/0b8f22f2-b1e6-48cd-8608-f576ad80e04b">

## 郵便番号で識別される住所のワード ブレークのずれの総計
[先のページ](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AddressDictionayforEDM.md)で作成した郵便番号で識別される住所で、
どれくらいのワード ブレークのぶれが発生するかについて、Microsoft 365 のワード ブレーカーを利用したカスタムのアプリによる分析結果では、以下の通りとなります。特に丁番号の有無によらずぶれないものが 88% あるものの、残り 12% の住所では、先の丁番号がない Ⓐ のパターン、丁番号が一つ入る Ⓑ のパターン、1-2-3 の丁番号、番地番号、号番号が入る Ⓒ のパターンで、ワード ブレークが異なるものが存在します。以下の表で、Ⓐ≠Ⓑ≠Ⓒ≠Ⓐ となるのが、完全不一致、Ⓐ≠Ⓑ=Ⓒ が不一致 A、Ⓐ=Ⓒ≠Ⓑ が不一致 B、Ⓐ=Ⓑ≠Ⓒ を不一致 C としています。参考: [分析結果のExcel](https://github.com/YoshihiroIchinose/E5Comp/blob/main/WB/%E4%BD%8F%E6%89%80%E8%BE%9E%E6%9B%B8_d_ana_M365.xlsx)

<img width="310" alt="image" src="https://github.com/YoshihiroIchinose/E5Comp/assets/66407692/241abd80-b024-485f-9b58-85ca3754103e">

## キーワード辞書での対処
12% の程の住所では、丁番号等の有無によりワード ブレーク位置が異なるバリエーションが存在することになりますが、そういったワード ブレークの位置が異なる
バリエーションも含めて検知をしたい場合、キーワード辞書に、あらかじめワード ブレークを想定したキーワード登録をします。
その一つの方法は、検知したいワード単位にキーワードを分割し、前・後ろも含めて"\_"で連携したものを登録しておくことです。
先の例では、通常の Ⓐ のパターンのフラットな住所の記述に加えて、Ⓑ および Ⓒ を想定した以下の文字列を合わせて登録することになります。

愛知県愛西市南河田町    
\_愛知_県_愛_西_市_南河_田_町_   
\_愛知_県_愛_西_市_南河_田町_   

なお、上記の文字列は、Word マッチのためにワードブレーク処理がされると"_"が除去され、想定した通りのワードブレークがなされます。

カスタムのアプリによるワード ブレーク動作の結果   
<img width="458" alt="image" src="https://github.com/YoshihiroIchinose/E5Comp/assets/66407692/bb557503-8bb2-4e2d-9d16-04d5f1ac1e0e">

## Microsoft 365 のワード ブレークのずれを反映させた住所辞書
[先のページ](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AddressDictionayforEDM.md)で生成した住所辞書を元に、"\_" を挿入する手法で、ワードブレーク位置が異なる住所を追加した日本の住所の辞書が[こちら](https://github.com/YoshihiroIchinose/E5Comp/blob/main/WB/JPAddressDicwithVariations.txt)です。
この住所辞書は、UTF-16 のエンコードで、3.54MB のサイズとなっていますが、PowerShell で取り込むことで、圧縮後 1MB のキーワード辞書のサイズ制限に収まるものとなります。

## UI を通じたキーワード辞書の更新について
左記の通り、"\_"で区切る手法により、新規に UI や PowerShell でキーワード辞書を定義する際は問題がないですが、UI で、キーワード辞書を更新することはできません。これは、既存で登録されたキーワードを読み込む際、UI 上では、先の "\_" を削除した文字列が読み込まれて表示されるためで、そのまま更新して保存すると、"\_" で分けたもののも、分けなかったものも、すべて同じ文字列として保存され、ワード ブレークの明示的な指定が失われてしまいます。そのため、こういった辞書を管理する場合には、PowerShell を通じて、辞書全体を更新するような運用が必要となります。

## PowerShell を通じた住所辞書の取り込み
[Exchange Online PowerShell](https://learn.microsoft.com/ja-jp/powershell/exchange/connect-to-exchange-online-powershell?view=exchange-ps) を用いることで、住所の辞書を以下のような PowerShell のコマンドで取り込むことができます。
```
Connect-IPPSSession   
$fileData = [System.IO.File]::ReadAllBytes("C:\WB\JPAddressDicwithVariations.txt")

#新規作成の場合
New-DlpKeywordDictionary -Name "JPAddress"  -Description "郵便番号ベースの日本の住所" -FileData $fileData

#更新の場合
Set-DlpKeywordDictionary -Identity "JPAddress" -FileData $fileData
```
