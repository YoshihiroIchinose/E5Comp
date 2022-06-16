# EDM のデータ アップロード用に全角の英数字・記号を半角に変換する
こちらの PowerShell のサンプルで、EDM のデータ アップロード用の CSV ファイルを事前処理し、全角の英数字・記号を半角に変換することができます。

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
