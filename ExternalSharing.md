# 社外とのセキュアなファイル共有方法   
PPAP に代わり Office 365 で利用可能なセキュアなファイル共有方法をまとめたものです。②～⑤ については、AIP を利用したファイル暗号化が伴います。社外ユーザーが AIP で保護されたファイルを開く場合の条件については、[こちら](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AIP_FAQ.MD)を参照の事。

|  方法 | 概要 | 社外アカウントの必要有無 | ファイルの二次利用制限 | 対応ファイル |
|:---|:---|:---|:---|:---|
| ①SharePoint Online / OneDrive for Business によるパスワード付き匿名リンク | ファイル アクセス用の固有 URL 共有して、URL を知っている人に対して、ユーザー認証不要でファイルを共有する方法。ファイル アクセス用のパスワードを設定することも可能。Office ファイルであればダウンロード禁止の設定も可能。 | 不要 | 不可 | 匿名リンクは特にファイル制限なし。ダウンロード禁止は、Office ファイル限定。<br> ダウンロード禁止の場合 Office for the Web の利用となるため、Excel ファイルは最大 100MB まで対応。<br>[SharePoint でのブックのファイル サイズの制限](https://support.microsoft.com/ja-jp/office/9e5bc6f8-018f-415a-b890-5452687b325e)|
| ②個別に AIP 保護したファイルをメールに添付して送付 | 受け手側のメールアドレスを指定した、AIP による Office ファイルの暗号化を実施 | 必要 | 可能 | ・Word(doc, docx, docm、dot、dotx dotm)<br>・Excel(xls, xlsx, xlsm, xlt, xltx, xltm, xlsb)<br>・PowerPoint(ppt, pptx, pptm,potx, potm, pps, ppsx, ppsm)<br>・Visio (vsdm,.vsdx,vssm,vssx,vstm,vstx)<br>・XPS<br>・PDF<br>・pFile (bpm, gif, jfif, jpe, jpeg, jpg, jt, png, tif, tiff, txt,xla, xlam, xml)<br>[参考: 保護がサポートされているファイルの種類](https://docs.microsoft.com/ja-jp/azure/information-protection/rms-client/clientv2-admin-guide-file-types#file-types-supported-for-protection)<br> Outlook/Exchange Online の場合添付ファイルの最大サイズは 150MB、Outloo on the Web の場合、112MB(150MB から 33% のオーバーヘッドを考慮) [メッセージの制限](https://docs.microsoft.com/ja-jp/office365/servicedescriptions/exchange-online-service-description/exchange-online-limits#message-limits)|
| ③SharePoint Online で IRM ライブラリを利用した B2B 共有をする | 個別に社外ユーザーを招待し、サイト内でダウンロードした際 AIP 保護が自動適用される IRM ライブラリを通じたファイル共有をする。IRM ライブラリの設定でファイル ダウンロード後の有効期間の設定も可能。 | 必要<br>・Office 365 テナントのID<br>・B2B 統合機能が有効であればワンタイム パスワードによる認証も可能 | 可能 | ・Word、 Excel、PowerPoint の Open XML 形式および旧形式<br>・InfoPath<br>・XPS<br> [参考:リストとライブラリに対する IRM の動作](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/apply-irm-to-a-list-or-library?view=o365-worldwide#how-irm-works-for-lists-and-libraries)|
| ④Message Encryption | Outlook/Exchange Online のメール オプションの暗号化設定で、転送不可を選択した上で、暗号化に対応した Office ファイルを添付して送付する。Office ファイルだけではなく、管理者設定で、PDF の保護にも対応可能。Advanced Message Encrpytion のテンプレート設定で、相手が Office 365 ユーザーであっても直接暗号化ファイル・メールを送らず OME (Office 365 Message Encryption)ポータルにリダイレクトさせることも可能。 | 不要<br>・受け手が Office 365 テナントであれば、AIP 暗号化された状態でメールおよび添付ファイルが送信される<br>・受け手が Office 365 テナントではない場合、OME (Office 365 Message Encryption)ポータルにリダイレクトされ、専用サイトでワンタイム パスコード認証した上で、コンテンツをブラウザで表示<br>・対応したファイルをダウンロードした場合には、AIP で暗号化された状態となる<br> | 可能 | ・Word(doc, docx, docm、dot、dotx dotm)<br>・Excel(xls, xlsx, xlsm, xlt, xltx, xltm, xlsb, xla, xlam)<br>・PowerPoint(ppt, pptx, pptm, pot, potx, potm, pps, ppsx, ppsm, thmx)<br>・IntoPath(xsn)<br>・XPS<br>・PDF(有効化設定必要)<br>[参考:メール メッセージでの IRM の使用方法](https://support.microsoft.com/ja-jp/office/bb643d33-4a3f-4ac7-9770-fd50d95f58dc)<br>メール本文およぶ添付ファイル含めて最大 25MB まで。<br>[OME で送信できるメッセージのサイズに制限はありますか?](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/ome-faq?view=o365-worldwide#ome--------------------------)<br>|
| ⑤Defender for Cloud Apps によるセッション制御 | Azure AD と SSO 連携しているなど認証を構成可能な管理された SaaS アプリへの Web アクセスを、Defender for Cloud Apps のリバース プロキシ経由にして、ファイル ダウンロード時に AIP の秘密度ラベルを適用する。既存の SharePoint Online サイトでも構わないが、社外ユーザーがアクセスできるファイル共有サイトが必要。 | 必要 | 可能 | ・Word( docm、docx、dotm、dotx)<br>・Excel(xlam、xlsm、xlsx、xltx)<br>・PowerPoint(potm、potx、ppsx、ppsm、pptm、pptx)<br>・PDF<br>[参考: ダウンロード時にファイルを保護する](https://docs.microsoft.com/ja-jp/defender-cloud-apps/session-policy-aad#protect-download)<br> AIP 保護できるのは最大で 50MB のファイルまで。[ファイルにラベルを直接適用する](https://docs.microsoft.com/ja-jp/defender-cloud-apps/azip-integration#how-to-integrate-azure-information-protection-with-defender-for-cloud-apps) |

 # ①SharePoint Online / OneDrive for Business によるパスワード付き匿名リンク
 ## テナントの要件
 SharePoint Online /OneDrive for Business で匿名リンクの利用が許可されている必要がある
 ## 設定概要
 SharePoint Online /OneDrive for Business にファイルをアップロードした後、ファイルのメニューから、共有->"リンクを知っているすべてのユーザー"を選んだ上で、パスワードの設定行ったり、"編集を許可する"からチェックを外した上で、"ダウンロードを禁止する"を有効化する。   
 <img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/FS_1.png">   
 
 # ②個別に AIP 保護したファイルをメールに添付して送付
 ## テナントの要件
 AIP でのファイル保護のために Office 365 E3 もしくは Azure Informatino Protection P1 を含むいずれかのライセンスが必要   
 ## 設定概要
 Office の Information Rights Management の機能や、Unified Label AIP Client を用いて、社外ユーザーのメールアドレスを指定した権限設定を行い、事前に AIP 保護したファイルを生成しておき、そのファイルをメールに添付して送付する。  
  
 # ③SharePoint Online で IRM ライブラリを利用した B2B 共有をする
 ## テナントの要件
 ・SharePoint Online で外部招待が許可されている必要がある    
 ・IRM ライブラリの利用に Office 365 E3 もしくは Azure Informatino Protection P1 を含むいずれかのライセンスが必要   
 ## 設定概要
 対象となる SharePoint Online のサイトのライブラリの設定で、IRM (Information Rights Managment)の設定を有効化した上で、対応したファイルをアップロードする。この設定により、ファイル ダウンロード時本人しか開けない権限設定で保護されたファイルが都度生成される。オプション設定により、対応しないファイルのアップロードをブロックすることや、ファイル ダウンロード後の有効期間、追加で付与するグループの権限なども設定可能。
 <img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/IRM1.png">
     
 # ④Message Encryption
 ## テナントの要件
 ・標準の Message Encrption の利用は Office 365 E3 もしくは Azure Informatino Protection P1 を含むいずれかのライセンスが必要   
 ・Advanced Message Encryption は、Microsoft 365 E5 / Office 365 E5 / E5 Complinace / IP&G のいずれかのライセンス必要   
 ## 設定概要
Exchange Online に接続した Outlook で、新規メールを作成時、オプションの暗号化設定より、転送不可を選択した上で、ファイルを添付し送付する。受け手が Office 365 テナントであれば、AIP 暗号化された状態でメールおよび添付ファイルが送信される。受け手が Office 365 テナントではない場合、OME (Office 365 Message Encryption)ポータルにリダイレクトされ、専用サイトでワンタイム パスコード認証した上で、コンテンツをブラウザで表示できる。OME ポータルから対応したファイルをダウンロードした場合には、AIP で暗号化された状態となる。
 <img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/FS_2.png">
 
 # ⑤Defender for Cloud Apps
 ## テナントの要件
 Defender for Cloud Apps の利用に、Defender for Cloud Apps を含むライセンスが必要  
 条件付きアクセス制御のために、Azure Active Directory P1 を含むライセンスが必要   
 AIP でのファイル保護のために Azure Informatino Protection P1 を含むライセンスが必要   
 ## 設定概要
 以下のような対応アプリを中心に、テナント管理者により Azure AD との SSO 設定を行った上で、Azure AD 条件付きアクセスで、特定の条件下でのセッション制限を有効化する。Defender for Cloud Apps の設定で、セッション ポリシーを定義し、どういった制御を行うか定義する。これら設定で、セッション制限下で、対応したファイルのダウンロード時、秘密度ラベルもしくは、ダウンロードしたユーザーに限定した権限設定でファイルを動的に AIP 保護することができる。
  <img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDCA1.png">   
  
セッション制御対応済みアプリ
- AWS
- Azure DevOps (Visual Studio Team Services)
- Azure portal
- ボックス
- Concur
- CornerStone on Demand
- DocuSign
- ドロップボックス
- Dynamics 365 CRM (プレビュー)
- Egnyte
- Exchange Online
- GitHub
- Google Workspace
- HighQ
- JIRA/Confluence
- OneDrive for Business
- LinkedIn Learning
- Power BI
- Salesforce
- ServiceNow
- SharePoint Online
- Slack
- Tableau
- Microsoft Teams (プレビュー)
- Workday
- Workiva
- Workplace by Facebook
- Yammer (プレビュー)   

なお上記以外のアプリでも、SAML 2.0 や、Open ID Connect で認証方法を変更できるのであれば、対応可能。   
[参考:サポートされているアプリとクライアント](https://docs.microsoft.com/ja-jp/defender-cloud-apps/proxy-intro-aad#featured-apps)
