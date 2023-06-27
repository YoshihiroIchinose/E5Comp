# アクティビティ エクスプローラのログの CSV への書き出しサンプル
新たに追加された Export-ActivityExplorerData のコマンドを利用して、最大 30 日分、5 万行 (5,000 行x 10 回)のログを取得するサンプルです。素の単体実行のコマンドでは、
一回につき最大 5,000 行までの取得制限があるので、5,000 行を超えたログの取得では、結果と合わせて返される WaterMark 変数を次回の PageCookie として指定して、
複数回に分けた取得が必要です。なお -outputformat csv の形式でも、CSV 出力が可能ですが、この場合、列名が出力されないという制限があり、
一方で -ouputformat json にすると、ログの種類ごとに記録されている列名のセットが異なるので、CSV 形式への加工が難しくなります。
このサンプルでは、 -ouputformat json を利用しつつ、列の全体を分析した上で、CSV 形式に出力することで、出力結果の CSV の先頭行に列名が入った出力が可能となっています。

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
	$d=Export-ActivityExplorerData -OutputFormat json -StartTime $s -EndTime $e -PageSize 5000 -PageCookie $watermark
	#$d=Export-ActivityExplorerData -OutputFormat json -StartTime $s -EndTime $e -PageSize 5000 -PageCookie $watermark -filter1 $filter1 -filter2 $filter2 -$filter3 $filter3
	$output+=convertfrom-json $d.ResultData
      "ループ $i :"+$output.count+"行"
	if($d.LastPage){break;}
      $watermark=$d.WaterMark
}

# ログの種類ごとに集計して、列の種類を把握する
$FieldName=@()
$activityTypes=$output|Group-Object ActivityId
foreach($activity in $activityTypes){
    $FieldsInJson=$activity.Group[0]|get-member -type NoteProperty
    foreach($f in $FieldsInJson){
      if($FieldName.Contains($f.Name) -eq $false) { $FieldName+=$f.Name}
    }
}

# 列の種類ごとに出力内容を ScriptBlock で定義する。Json 形式のデータは、パースして、"" で閉じる。
$Fields=@()
foreach($f in $FieldName){
    $sb=[scriptblock]::Create('$att=$row.'+$f+';if($att.GetType().Name -eq "Object[]" -or $att.GetType().Name -eq "PSCustomObject"){$att="`""+(ConvertTo-Json -Compress -Depth 10 $att)+"`""}$att')
    $Fields+=@{Name=$f;Expression=$sb}
}

# データを加工
$csv=@();
foreach($row in $output){
    $data=$row|Select-Object -Property $Fields
    $csv+=$data
 }
 
 # Excel ファイルに書き出す
$csv|Export-Csv -Path ($OutputFile) -NoTypeInformation -Encoding UTF8
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
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AE.png">
