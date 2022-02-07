# 社外とのセキュアなファイル共有方法   


|  方法 | 概要 | オプション | テナント要件 | 社外アカウントの必要有無 | ファイルの二次利用制限 |   
|:---|:---|:---|:---|:---|:---|
| SharePoint Online / OneDrive for Business によるパスワード付き匿名リンク | URL を知っている人に対して、ユーザー認証不要でファイルを共有する方法 | Office ファイルであればダウンロード禁止の設定も可能 | SharePoint Online /OneDrive for Business で匿名リンクの利用が許可されている必要がある | 不要 | 不可 |
| SharePoint Online で IRM ライブラリを利用した B2B 共有をする | 個別に社外ユーザーを招待し、サイト内でダウンロードした際 AIP 保護が自動適用される IRM ライブラリを通じたファイル共有をする | IRM ライブラリの設定でファイル ダウンロード後の有効期間の設定も可能 | ・SharePoint Online /OneDrive for Business で外部招待が許可されている必要がある<br>・IRM ライブラリの利用に Office 365 E3 / EMS E3 / AIP P1 が必要 |  必要<br>・Office 365 テナントのID<br>・B2B 統合機能が有効であればワンタイム パスワードによる認証も可能 | 可能 |
| Message Encryption | Outlook/Exchange Online のメール オプションの暗号化設定で、転送不可を選択した上で、暗号化に対応した Office ファイルを添付して送付する | ・Office ファイルだけではなく、管理者設定で、PDF の保護にも対応可能<br>・Advanced Message Encrpytion のテンプレート設定で、相手が Office 365 ユーザーであっても直接暗号化ファイル・メールを送らず OME (Office 365 Message Encryption)ポータルにリダイレクトさせることも可能。 | ・標準の Message Encrption の利用は Office 365 E3 / EMS E3 / AIP P1<br>・Advanced Message Encryption は、Microsoft 365 E5 / Office 365 E5 / E5 Complinace / IP&G | 不要<br>・受け手が Office 365 テナントであれば、AIP 暗号化された状態でメールおよび添付ファイルが送信される<br>・受け手が Office 365 テナントではない場合、OME (Office 365 Message Encryption)ポータルにリダイレクトされ、専用サイトでワンタイム パスコード認証した上で、コンテンツをブラウザで表示<br>・対応したファイルをダウンロードした場合には、AIP で暗号化された状態となる<br> | 可能 | 


