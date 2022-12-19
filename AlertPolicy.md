# アラート ポリシーを利用して監査ログに記録された操作に応じてアラートを発令する
Purview Portal の[アラート ポリシー](https://learn.microsoft.com/ja-jp/microsoft-365/compliance/alert-policies?view=o365-worldwide)では、
DLP の設定などに応じて、どういった状況でアラートを生成し、管理者への通知を行うかの設定が確認できます。
またこのアラート ポリシーの仕組みを用いて、監査ログに記録される各種操作に応じたアラートの発令も設定することができます。
ただし、[Web UI](https://compliance.microsoft.com/alertpoliciesv2) からでの設定では、任意の監査ログの Operation を選択することができず、やや汎用性にかけます。
一方で PowerShell で用意されている [New-ProtectionAlert](https://learn.microsoft.com/ja-jp/powershell/module/exchange/new-protectionalert?view=exchange-ps) のコマンドレットを利用すれば、任意の監査ログの Operation に応じたアラートの定義が可能です。

## ライセンス
一定時間の閾値や、7 日間のアノマリーで閾値を設定する場合、MDO P2、E5、E5 Complinace、eDiscovery and Audit のいずれかのライセンスが必要となる。時間の範囲は最も短くて 60 分、操作回数の閾値の下限は 3。<br>
特に閾値を設けない場合でも、上位版のライセンスがあれば、1 分間のインターバルで同様の操作は一つのアラートに集計され、上位版のライセンスがなければ、15 分間のインターバルで同様の操作が集計されて、1 つのアラートとなる。<br>
[参考: アラートの集計](https://learn.microsoft.com/ja-jp/microsoft-365/compliance/alert-policies?view=o365-worldwide#alert-aggregation)


## 秘密度ラベル操作に応じたアラート例

### 秘密度ラベルを削除する度にアラートを生成する。
````
Connect-IPPSSession
New-ProtectionAlert -Category DataLossPrevention -Name SensitivityLabelRemoved -NotifyUser admin@xxxx.onmicrosoft.com  -ThreatType Activity -Operation SensitivityLabelRemoved -AggregationType none -NotificationCulture ja-JP
````

### 1 時間の中で 3 回以上、秘密度ラベルを変更するとアラートを生成する。
````
Connect-IPPSSession
New-ProtectionAlert -Category DataLossPrevention -Name SensitivityLabelUpdated -NotifyUser admin@mxxx.onmicrosoft.com  -ThreatType Activity -Operation SensitivityLabelUpdated -NotificationCulture ja-JP -Threshold 3 -TimeWindow 60
````
### アラート通知
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/alert/Alert01.png" height="400px">

### アラート詳細
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/alert/Alert02.png" height="400px">

### 補足
秘密度ラベルの手動操作は、下記の RecodTyped で記録され、それぞれ SharePoint Online 上の Office for the Web、M365 Apps、AIP Client の操作が記録される。<br>
6 SharePointFileOperation    
83 SensitivityLabelAction    
94 AipSensitivityLabelAction    
[参考: Office 365 監査ログでの RecordType について](https://github.com/YoshihiroIchinose/E5Comp/blob/main/RecordTypes.md)    

M365 Apps や AIP クライアントのいずれの RecordType であっても、以下の共通の Operation が記録されるため、サンプルのような
秘密度ラベルの削除であれば、SensitivityLabelRemoved、
秘密度ラベルの変更であれば、SensitivityLabelUpdated の Operation を対象にしたアラートを設定すれば、
M365 Apps や AIP クライアントかによらず、アラートの対象とすることができる。<br>
SensitivityLabelApplied    
SensitivityLabelUpdated    
SensitivityLabelRemoved    
SensitivityLabelPolicyMatched    
SensitivityLabeledFileOpened    

一方で Office for the Web で秘密度ラベルを操作した際には、6 SharePointFileOperation の RecordType で M365 Apps や AIP Client とは
違った以下の名称の Operation で操作が記録されるため、これらを対象にする場合には、別途アラートの定義を行う必要がある。    
FileSensitivityLabelApplied    
FileSensitivityLabelChanged    
FileSensitivityLabelRemoved    

### 制限事項
アラート ポリシーを利用して秘密度ラベルの操作を監視する場合においては、以下の制限事項がある。
1. ラベルの変更や削除などの操作に対してアラート設定は可能だが、ログが分かれておらず、ログの内容に応じたフィルタもできないため、秘密度ラベルのダウングレード操作のみを対象とすることはできない。
2. アラート対象となるユーザーをグループで限定しておくことはできない。アラートを設定する場合、テナント全体のユーザーに対して共通の閾値で設定することとなる。(ユーザーの属性などでフィルタ出来る可能性もあるが、一見したところ、Azure AD の属性や、セキュリティ グループなどでフィルタすることはできなさそう。)
3. アラートは発令し、アラートを確認した場合には、誰がいつその操作を何回行ったかは判別可能であるが、監査ログに記録されているものがそのまま記載されているわけではないため、どんなラベルからどう変更したかなどは判別できない。<br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/alert/Alert03.png"><br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/alert/Alert04.png" height="400px"><br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/alert/Alert05.png" height="400px"><br>

これらを踏まえると、アラートが発令された場合に調査を行いたければ、Activity Explorer の [UI](https://compliance.microsoft.com/dataclassification?viewid=activitiesexplorer) もしくは、[PowerShell で抜き出した情報](https://github.com/YoshihiroIchinose/E5Comp/blob/main/ActivityExplorerData.md)を元に詳細を分析することが望ましい。Acvitiy Explorer の情報では、元の秘密度ラベルや、現行の秘密度ラベルだけではなく、監査ログには記録されていない、LabelDowgarded や LabelUpgraded などのラベル イベントの種類が判定されているため、より効率的なフィルタや調査が可能となっている。<br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/alert/Alert06.png" height="400px">
