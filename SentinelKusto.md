# Sentinel で秘密度ラベルのダウングレード操作をアラートする
Microsoft Purview Information Protection のデータ コネクタで取り込んだ秘密度ラベルの操作のログを元に、
一定期間内に複数回秘密度ラベルのダウングレードがあった場合に、それらをアラートする仕組みを Kusto クエリで作成するサンプルです。

## ベースとなる Kusto クエリ
以下の Kusto クエリでは、MicrosoftPurviewInformationProtection のテーブルから、LableEventType が "LabelDowngraded" となる
ログを対象に絞り込んでいます。その上で、一定期間に複数回ラベルのダウングレードがあった場合のみアラートにしたいため、
ログの発生時間で、1 時間ごとの範囲でまるめた上で集計し、今回の例では、毎時 3 回以上操作があった場合をアラートの対象にしています。   

ただし、この簡便な集計方式の制約として、各時 0 分をまたいだケースでは積算されず、例えば、 14 時台に 1 回、15 時台に 2 回の
操作があったケースは、アラートの対象となりません。
また単純に summarize で集計した場合、各ログの詳細情報が失われてしまうため、集計前にファイル名、以前の秘密度ラベル、新しい秘密度ラベル、
ラベル変更時の理由、操作した時間を、bag_pack でプロパティ バッグにまとめた上で、集計の際、make_set で JSON 配列として連結してあります。
```
MicrosoftPurviewInformationProtection
| where LabelEventType == "LabelDowngraded" 
| extend LabelDetail = bag_pack("File",ObjectId,"OldLabel",OldSensitivityLabelId,"NewLabel",SensitivityLabelId,"Justification",JustificationText, "Time",TimeGenerated)
| summarize counts=count(), LabelDetails=make_set(LabelDetail) by UserId, length=bin(TimeGenerated, 60m)
| where counts >=3
```

## 秘密度ラベルの表示名にする
MicrosoftPurviewInformationProtection のログでは各秘密度ラベルは、GUID で記録されるため表示名で秘密度ラベルを確認したい場合、
GUID から表示名に変換するマッピング テーブルを用意する必要があります。これは以下の公式サイトでも紹介されています。
[秘密度ラベルの表示名をマッピングする](https://learn.microsoft.com/en-us/azure/sentinel/connect-microsoft-purview#known-issues-and-limitations)

## 秘密度ラベルの情報を PowerShell で取得する
上記サイトで紹介されているマッピング テーブルを作成する場合には、Exchange Online PowerShell を用いて、
定義済みの秘密度ラベルの情報を参照することで、得られます。以下のサンプルは、PowerShell を用いて、
Kusto クエリに張り付けやすい JSON 形式で、秘密度ラベルの GUID と表示名の情報を出力するサンプルです。
```
Connect-IPPSSession
$labels=Get-Label|?{$_.ContentType.IndexOf("File") -ne -1}
$a=@{}
$labels|%{$a.add($_.Guid.ToString(),$_.Name)}
$a|convertto-json
```
## 上記 PowerShell のコマンドで、以下ような JSON 形式の出力結果が得られます。
```
{
    "30b5e379-19c1-4793-8850-770933e0bf5e":  "業務外",
    "c7861feb-52bc-4794-9c51-6e7f088ee93c":  "誰でも開ける暗号化",
    "2605be26-f499-4a91-993d-520e8650d0c6":  "社外秘",
    "5759c9fa-cc5c-4c01-9858-a0e53b65e13e":  "業務"
}
```
## 秘密度ラベルを表示名に変換する Kusto クエリ例
先の PowerShell で得られた出力結果を貼り付け、GUID で記録していた部分を、表示名に置き換える処理を入れることで、Sentinel のアラーとで、
秘密度ラベルを表示名で確認することができるようになります。なお JSON の Kusto クエリの貼り付けに当たっては、各 GUID と表示名を
マッピングする行の前後を、'(シングル クォーテーション)で囲むこと、JSON 定義の後は、;(セミコロン)を入れて、空白行を入れずに
続くクエリを記載する必要がある点に注意します。
```
let labelsMap = parse_json('{'
    '"30b5e379-19c1-4793-8850-770933e0bf5e":  "業務外",'
    '"c7861feb-52bc-4794-9c51-6e7f088ee93c":  "誰でも開ける暗号化",'
    '"2605be26-f499-4a91-993d-520e8650d0c6":  "社外秘",'
    '"5759c9fa-cc5c-4c01-9858-a0e53b65e13e":  "業務"'
 '}');
MicrosoftPurviewInformationProtection
| where LabelEventType == "LabelDowngraded"
| extend SensitivityLabelName = iif(isnotempty(SensitivityLabelId), tostring(labelsMap[tostring(SensitivityLabelId)]), "")
| extend OldSensitivityLabelName = iif(isnotempty(OldSensitivityLabelId), tostring(labelsMap[tostring(OldSensitivityLabelId)]), "")
| extend LabelDetail = bag_pack("File",ObjectId,"OldLabel",OldSensitivityLabelName,"NewLabel",SensitivityLabelName,"Justification",JustificationText, "Time",TimeGenerated)
| summarize counts=count(), LabelDetails=make_set(LabelDetail) by UserId, length=bin(TimeGenerated, 60m)
| where counts >=3
```
