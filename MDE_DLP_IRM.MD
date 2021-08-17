#各サービスでのログ

| 操作 | Defender for Endpoint Timeline | Endpoint DLP の Activity Explorer | Insider Risk Management での Activity Explorer |
| ---- | ------------ | ------------- | ------------- |
| 1.社内ファイル サーバーからのファイル ダウンロード | 〇2 つのログ explorer.exe created file xxx.txt xxx.txt, file observed on host を記録 (条件：MDE の監視対象のファイル) | 〇 File created のログを記録 (条件: Endpoint DLP の監視対象のファイル) | 〇 Sensitive File created のログを記録 (条件: 何等かの機密情報が含まれている場合)|
| 2.ダウンロードしたファイルを WinRAR.exe などの圧縮ソフトで読み取り | × 特にログ記録なし | 〇 File accessed by unallowed app のログを記録 (条件: DLP ポリシーで指定された機密情報を含むファイルの参照で Unallowed apps に指定された exe からのアクセス) | 〇 Sensitive File Read のログを記録 (条件: 何等かの機密情報が含まれている場合で全 exe が対象)|
| 3.WinRar.exe などの圧縮ソフトから分割した RAR ファイルを生成 | 〇 WinRAR.exe created file xxx.part01.rar, WinRAR.exe created file xxx.part02.rar のログを記録 | 〇 File created xxx.part01.rar, File created xxx.part02.rar のログを記録 | × 特にログ記録なし |
| 4.外部の Web サイトに RAR ファイルをアップロード | × 特にログ記録なし | 〇 File copied to cloud のログを記録 (条件: Edge およびアドオンを入れた Chrome 経由)| 〇 File copied to cloud のログを記録 (条件: Edge およびアドオンを入れた Chrome 経由)|


## Endpoint DLP の監視対象
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


