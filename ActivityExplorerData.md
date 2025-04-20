# アクティビティ エクスプローラのログの CSV への書き出しサンプル
新たに追加された Export-ActivityExplorerData のコマンドを利用して、最大 30 日分、5 万行 (5,000 行x 10 回)のログを取得するサンプルです。素の単体実行のコマンドでは、
一回につき最大 5,000 行までの取得制限があるので、5,000 行を超えたログの取得では、結果と合わせて返される WaterMark 変数を次回の PageCookie として指定して、
複数回に分けた取得が必要です。現在は、CSV 形式で出力した場合でも列名が記録されるため、列名を取得するために JSON 形式でデータを取得して CSV に変換するなどのデータ加工は不要です。

# PowerShell スクリプト
## PowerShell を管理権限で立ち上げて事前に一度以下を実行
```
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module -Name ExchangeOnlineManagement
```
## Endpooint DLP のログをまとめて取得
```
#接続に利用するID / パスワード
$Id="xxx@xxx.onmicrosoft.com"
$Password = "xxxxx"
#変数
$date=get-date
$e=$date.ToUniversalTime().ToString("yyyy/MM/dd HH:mm:ss")
$s=$date.AddDays(-30).ToUniversalTime().ToString("yyyy/MM/dd HH:mm:ss")
$OutputFile=[System.Environment]::GetFolderPath("Desktop")+"\AEJ.csv"
#$filter1=@("Activity", "DLPRuleMatch")
#$filter2=@("Workload", "Exchange")
#$filter3=@("Policy", "DLPPolicyName")

#Credentialの生成
$SecPass=ConvertTo-SecureString -String $Password -AsPlainText -Force
$Credential= New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $Id, $SecPass

Connect-IPPSSession -credential $Credential

# 最大 5,000 行x 10 回でログを取得
$output=@();
$watermark="";
for($i = 0; $i -lt 10; $i++){
	$d=Export-ActivityExplorerData -OutputFormat csv -StartTime $s -EndTime $e -PageSize 5000 -PageCookie $watermark
	#$d=Export-ActivityExplorerData -OutputFormat csv -StartTime $s -EndTime $e -PageSize 5000 -PageCookie $watermark -filter1 $filter1 -filter2 $filter2 -$filter3 $filter3
	$output+=$d.ResultData
      "ループ $i :"+$d.RecordCount+"行"
	if($d.LastPage){break;}
      $watermark=$d.WaterMark
}
 
# Excel ファイルに書き出す
$output|out-file ($OutputFile) -Encoding UTF8
# 特定の列だけ書き出す場合
# $output|ConvertFrom-Csv|Select-Object -Property "Activity", "Happened", "User","FilePath","TargetFilePath","Application","DeviceName","Manufacturer","SerialNumber","Model" |Export-Csv ($OutputFile) -Encoding UTF8 -NoTypeInformation

```

## (おまけ) Excel を通じて CSV ファイルをテーブル フォーマットありの Excel ファイルに変換
```
$excel = new-Object -com excel.application
$excel.visible = $false
$book = $excel.Workbooks.open($OutputFile)
$book.ActiveSheet.ListObjects.Add(1,$book.ActiveSheet.Range("A1").CurrentRegion ,$null,1).Name = "テーブル1"
$book.SaveAs($OutputFile.replace(".csv",".xlsx"),51)
$book.close()
$excel.quit()
```
## Excel 出力結果例
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AE2.png">
