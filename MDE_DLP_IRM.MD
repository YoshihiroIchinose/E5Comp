# ファイル操作に関する各サービスでのログの違い
これらは、すべて MSSense.exe が実体である、Defender for Endpoint と同じセンサーを利用するが、監視の目的の違いから各サービスの UI で確認できるログについて違いがある。
- Defender for Endpoint: 主にサイバー攻撃を検知するための仕組みであり、プロセス、通信、レジストリなどに特化した監視を行う。
- Endpoint DLP: 広範にポリシーを設定してユーザー全体を監視し、端末における全般的な情報の持ち出しを監視・制御する仕組み。
- Insider Risk Managment: ファイル操作の絶対量や、退職予定、個別の指定などにより、リスクが高いと判定されたユーザーの操作を後日改めて定量的に分析する仕組み。ポリシーなどにはよらず機密情報が含まれるファイルを主に対象とする。

## 一連の情報漏えい操作時のログの出方
| 操作 | Defender for Endpoint Timeline | Endpoint DLP の Activity Explorer | Insider Risk Management での Activity Explorer |
| ---- | ------------ | ------------- | ------------- |
| 1.社内ファイル サーバーからのファイル ダウンロード | 〇2 つのログ<br> explorer.exe created file xxx.txt<br>xxx.txt, file observed on host を記録 <br>(条件：MDE の監視対象のファイル) | 〇 File created のログを記録 <br>(条件: Endpoint DLP の監視対象のファイル + ポリシーで指定された拡張子のファイル) | 〇 Sensitive File created のログを記録 <br>(条件: 何等かの機密情報が含まれている場合)|
| 2.ダウンロードしたファイルを WinRAR.exe などの圧縮ソフトで読み取り | × 特にログ記録なし | 〇 File accessed by unallowed app のログを記録 <br>(条件: DLP ポリシーで指定された機密情報を含むファイルの参照で Unallowed apps に指定された exe からのアクセス) | 〇 Sensitive File Read のログを記録 <br>(条件: 何等かの機密情報が含まれている場合で全 exe が対象)|
| 3.WinRar.exe などの圧縮ソフトから分割した RAR ファイルを生成 | 〇 WinRAR.exe created file xxx.part01.rar, WinRAR.exe created file xxx.part02.rar のログを記録 | 〇 File created xxx.part01.rar, File created xxx.part02.rar のログを記録 | × 特にログ記録なし |
| 4.外部の Web サイトに RAR ファイルをアップロード | × 特にログ記録なし | 〇 FileUploadedToCloud のログを記録 <br>(条件: Endpoint DLP の監視対象のファイル + ポリシーで指定された拡張子のファイルで、Edge もしくはアドオンを入れた Chrome 経由でのアップロード)| 〇 File copied to cloud のログを記録 <br>(条件: Edge およびアドオンを入れた Chrome 経由)|

## Defender for Endpoint の FileCreated イベントで確認されたファイル拡張子
+ .4io
+ .7z
+ .asax
+ .aspx
+ .bat
+ .bmp
+ .cfg
+ .cmd
+ .com
+ .config
+ .cookie
+ .crx
+ .css
+ .csv
+ .dat
+ .dat_
+ .dll
+ .DLL
+ .dll_
+ .dll_DiP_STAGED
+ .dmp
+ .docx
+ .dotx
+ .exe
+ .EXE
+ .exe_
+ .exe_DiP_STAGED
+ .gif
+ .htm
+ .html
+ .HTML
+ .inf
+ .js
+ .lnk
+ .LNK
+ .msi
+ .msp
+ .mui
+ .node
+ .pdf
+ .pem
+ .png
+ .pptx
+ .ps1
+ .ps1xml
+ .psd1
+ .psm1
+ .rar
+ .SFX
+ .sst
+ .svg
+ .tmp
+ .TMP
+ .txt
+ .wer
+ .xar
+ .xlsb
+ .xlsx
+ .zip


## Endpoint DLP の監視対象ファイル
+ Word ファイル
+ PowerPoint ファイル
+ Excel ファイル
+ PDF ファイル
+ .csv ファイル
+ .tsv ファイル
+ .txt ファイル
+ .rtf ファイル
+ .c ファイル
+ .class ファイル
+ .cpp ファイル
+ .cs ファイル
+ .h ファイル
+ .java ファイル
+ .zip ファイル
+ .rar ファイル
+ .tar ファイル
+ .lzh ファイル

## Endpoint DLP の Activity Explorer で拡張子を指定することで FileUploadedToCloud が確認できたファイル拡張子 (おそらく全ファイル種別)
+ .cab
+ .csv
+ .docx
+ .dll
+ .exe
+ .jpg
+ .js
+ .lzh
+ .mp4
+ .pdf
+ .pfile
+ .pjpg
+ .png
+ .ptxt
+ .rar
+ .svg
+ .tar
+ .txt
+ .zip


## Insider Risk Manament の Activity Explorer で Sensitive File Read および Sensitive File Created が確認できたファイル拡張子
+ .asd
+ .docx
+ .dotx
+ .xlsx
+ .csv
+ .pdf
+ .pptx
+ .txt
+ .xlsx

## Insider Risk Manament の Activity Explorer で File Upload To Cloud が確認できたファイル拡張子 (おそらく全ファイル種別)
+ .cab
+ .csv
+ .docx
+ .dll
+ .exe
+ .jpg
+ .js
+ .lzh
+ .mp4
+ .pdf
+ .pfile
+ .pjpg
+ .png
+ .ptxt
+ .rar
+ .svg
+ .tar
+ .txt
+ .xlsx
+ .zip
