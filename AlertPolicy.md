# アラート ポリシーを利用して監査ログに記録された操作に応じてアラートを発令する
Purview Portal の[アラート ポリシー](https://learn.microsoft.com/ja-jp/microsoft-365/compliance/alert-policies?view=o365-worldwide)では、
DLP の設定などに応じて、どういった状況でアラートを生成し、管理者への通知を行うかの設定が確認できます。
またこのアラート ポリシーの仕組みを用いて、監査ログに記録される各種操作に応じたアラートの発令も設定することができます。
ただし、[Web UI](https://compliance.microsoft.com/alertpoliciesv2) からでの設定では、任意の監査ログの Operation を選択することができず、やや汎用性にかけます。
一方で PowerShell で用意されている New-ProtectionAlert のコマンドレットを利用することで、任意の監査ログの Operation に応じたアラートの定義可能です。

## ライセンス
一定時間の閾値を設定する場合、E5、MDO P2、eDiscovery and Auditのいずれかのライセンスが必要となる。時間の範囲は最も短くて 60 分、操作回数の閾値の下限は 3。

# 秘密度ラベル操作に応じたアラート例

## サンプル
秘密度ラベルを削除する度にアラートを生成する。
````
Connect-IPPSSession
New-ProtectionAlert -Category DataLossPrevention -Name SensitivityLabelRemoved -NotifyUser admin@xxxx.onmicrosoft.com  -ThreatType Activity -Operation SensitivityLabelRemoved -AggregationType none -NotificationCulture ja-JP
````

1 時間の中で 3 回以上、秘密度ラベルを変更するとアラートを生成する。
````
Connect-IPPSSession
New-ProtectionAlert -Category DataLossPrevention -Name SensitivityLabelUpdated -NotifyUser admin@mxxx.onmicrosoft.com  -ThreatType Activity -Operation SensitivityLabelUpdated -NotificationCulture ja-JP -Threshold 3 -TimeWindow 60
````

## 補足
秘密度ラベルの手動操作は、SharePoint Online 上の Office for the Web か、M365 APpsの RecordType で記録される。<br>
83 SensitivityLabelAction    
94 AipSensitivityLabelAction    
[参考: Office 365 監査ログでの RecordType について](https://github.com/YoshihiroIchinose/E5Comp/blob/main/RecordTypes.md)    
ただし、いずれの RecordType であっても、以下の共通の Operation が記録されるため、サンプルのように秘密度ラベルの削除であれば、SensitivityLabelRemoved、
秘密度ラベルの変更であれば、SensitivityLabelUpdated の Operation を対象にしたアラートを設定すれば、秘密度ラベルをどこで操作したかによらず、
アラートの対象とすることができる。
SensitivityLabelApplied    
SensitivityLabelUpdated    
SensitivityLabelRemoved    
SensitivityLabelPolicyMatched    
SensitivityLabeledFileOpened    
