# EDM のデータ アップロード用に全角の英数字・記号を半角に変換する
こちらの PowerShell のサンプルで、EDM のデータ アップロード用の CSV ファイルを事前処理し、全角の英数字・記号を半角に変換することを目的に、半角カナを全角カナに変換します。変換元ファイルは、BOM 付き UTF-8、UTF-16 LE、UTF-16 BE のいずれかである必要があり、BOM なし UTF-8 は NG です。既定のファイル出力結果は、UTF-16 LE になります。

## Normalize メソッドを利用する方法
以下のサンプルは Normalize メソッドを利用して行ごとに一括正規化します。
```
$source="E:\WorkData\Comp\EDM_Test\DB\EDM_CustomerDB.csv"
$target="E:\WorkData\Comp\EDM_Test\DB\EDM_CustomerDB_c.csv"

$output=@()
$lines=Get-Content $source
foreach($line in $lines){
   $output+=$line.Normalize([System.Text.NormalizationForm]::FormKC)
}
$output|out-file $target
```
## 実行結果
### 変換前
```
ABCabc
ＡＢＣａｂｃ
０３－３３３３－３３３３
03-3333-3333
品川 駅
品川　駅
#$%&
＃＄％＆
アイウエオ
ァィゥェォ
ｧｨｩｪｫ
パピプペポ
ﾊﾟﾋﾟﾌﾟﾍﾟﾎﾟ
ダヂヅデド
ﾀﾞﾁﾞﾂﾞﾃﾞﾄﾞ
ャュョ
ｬｭｮ
```
### 変換後
```
ABCabc
ABCabc
03-3333-3333
03-3333-3333
品川 駅
品川 駅
#$%&
#$%&
アイウエオ
ァィゥェォ
ァィゥェォ
パピプペポ
パピプペポ
ダヂヅデド
ダヂヅデド
ャュョ
ャュョ
```

## 愚直に文字置換する方法
以下のサンプルは愚直に 1 文字 1 文字置き換えるコードの例です。ただし、こちらの場合は、半角カナを考慮していません。半角カナを置き換える場合、濁音・半濁音が独立した文字となっており、2 文字 -> 1 文字の置換となるため、実装の工夫が必要です。
```
$source="E:\WorkData\Comp\EDM_Test\DB\EDM_CustomerDB.csv"
$target="E:\WorkData\Comp\EDM_Test\DB\EDM_CustomerDB_c.csv"

function ConvertTo-SingleBytes($str){
$chars=$str.ToCharArray()
$out = [char[]]::new($str.Length)
$count=0
foreach($char in $chars){
    $out[$count]=switch -CaseSensitive($char){
    '１' {'1';break}
    '２' {'2';break}
    '３' {'3';break}
    '４' {'4';break}
    '５' {'5';break}
    '６' {'6';break}
    '７' {'7';break}
    '８' {'8';break}
    '９' {'9';break}
    '０' {'0';break}
    '　' {' ';break}
    'Ａ' {'A';break}
    'Ｂ' {'B';break}
    'Ｃ' {'C';break}
    'Ｄ' {'D';break}
    'Ｅ' {'E';break}
    'Ｆ' {'F';break}
    'Ｇ' {'G';break}
    'Ｈ' {'H';break}
    'Ｉ' {'I';break}
    'Ｊ' {'J';break}
    'Ｋ' {'K';break}
    'Ｌ' {'L';break}
    'Ｍ' {'M';break}
    'Ｎ' {'N';break}
    'Ｏ' {'O';break}
    'Ｐ' {'P';break}
    'Ｑ' {'Q';break}
    'Ｒ' {'R';break}
    'Ｓ' {'S';break}
    'Ｔ' {'T';break}
    'Ｕ' {'U';break}
    'Ｖ' {'V';break}
    'Ｗ' {'W';break}
    'Ｘ' {'X';break}
    'Ｙ' {'Y';break}
    'Ｚ' {'Z';break}
    'ａ' {'a';break}
    'ｂ' {'b';break}
    'ｃ' {'c';break}
    'ｄ' {'d';break}
    'ｅ' {'e';break}
    'ｆ' {'f';break}
    'ｇ' {'g';break}
    'ｈ' {'h';break}
    'ｉ' {'i';break}
    'ｊ' {'j';break}
    'ｋ' {'k';break}
    'ｌ' {'l';break}
    'ｍ' {'m';break}
    'ｎ' {'n';break}
    'ｏ' {'o';break}
    'ｐ' {'p';break}
    'ｑ' {'q';break}
    'ｒ' {'r';break}
    'ｓ' {'s';break}
    'ｔ' {'t';break}
    'ｕ' {'u';break}
    'ｖ' {'v';break}
    'ｗ' {'w';break}
    'ｘ' {'x';break}
    'ｗ' {'y';break}
    'ｚ' {'z';break}
    '！' {'!';break}
    '＃' {'#';break}
    '＄' {'$';break}
    '％' {'%';break}
    '＆' {'&';break}
    '＾' {'^';break}
    '￥' {'\';break}
    '＠' {'@';break}
    '；' {';';break}
    '：' {':';break}
    '，' {',';break}
    '．' {'.';break}
    '／' {'/';break}
    '＝' {'=';break}
    '～' {'~';break}
    '｜' {'|';break}
    '‘' {'`';break}
    '｛' {'{';break}
    '＋' {'+';break}
    '＊' {'*';break}
    '｝' {'}';break}
    '＜' {'<';break}
    '＞' {'>';break}
    '？' {'?';break}
    '＿' {'_';break}
    '﹣' {'-';break}
    '－' {'-';break}
    default {$char}
    }
    $count++
  }
  return [String]::new($out)
}

$output=@()
$lines=Get-Content $source
foreach($line in $lines){
   $output+=ConvertTo-SingleBytes $line
}
$output|out-file $target
```

## 実行結果
### 変換前
```
ABCabc
ＡＢＣａｂｃ
０３－３３３３－３３３３
03-3333-3333
品川 駅
品川　駅
#$%&
＃＄％＆
アイウエオ
ァィゥェォ
ｧｨｩｪｫ
パピプペポ
ﾊﾟﾋﾟﾌﾟﾍﾟﾎﾟ
ダヂヅデド
ﾀﾞﾁﾞﾂﾞﾃﾞﾄﾞ
ャュョ
ｬｭｮ
```
### 変換後
```
ABCabc
ABCabc
03-3333-3333
03-3333-3333
品川 駅
品川 駅
#$%&
#$%&
アイウエオ
ァィゥェォ
ｧｨｩｪｫ
パピプペポ
ﾊﾟﾋﾟﾌﾟﾍﾟﾎﾟ
ダヂヅデド
ﾀﾞﾁﾞﾂﾞﾃﾞﾄﾞ
ャュョ
ｬｭｮ
```
