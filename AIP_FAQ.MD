# Azure Information Protection で保護されたファイルを外部ユーザーと共有する
Azure Information Protection では、自社テナントだけではなく、以下のような社外アカウントを利用するユーザーとも暗号化されたファイルの共有が可能となっています。 ただし、ファイルを共有しようとするユーザーからは完全にコントロールすることが難しい、各種前提条件もあり、あまり活用が進んでいないかと思われます。本ページでは、利用可否の判断をより具体的に行ってもらうことを目的に、これら前提条件や注意事項を明確にします。    

## 権限によらないテスト用ファイル
以下のファイルは、任意の認証済みユーザー (AuthenticatedUsers) に権限を付与して暗号化保護したファイルとなっており、どの外部ユーザーでも権限関係なしに開けるはずです。権限を抜きにして、ファイルを開ける環境かどうかテストする場合にお使いください。   
[Word ファイル](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AIP/AnyoneProtection.docx): M365 Apps もしくは Office + AIP UL Office アドインでの参照が必要   
[PDF ファイル](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AIP/AnyoneProtection.pdf): Edge もしくは AIP Viewer での参照が必要   
[PPNG ファイル](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AIP/AnyoneProtection.ppng): AIP Viewer での参照が必要   

## AIP で認証に利用できるアカウント
A. 外部の Office 365 テナントのアカウント  
B. [Azure Information Protection](https://aka.ms/rms-signup) のサイトにて会社のメール アドレスを用いて個別にサインアップしたアカウント<br>
(コンシューマー向けのメール ドメインを持つメール アドレスの利用は不可で結果的に Azure AD Free のアカウントとなる)  
C. Azure AD B2B の仕組みを使ったメール アドレス＋ワンタイム パスコードを利用する外部ユーザー アカウント  
D. コンシューマーの Microsoft Account<br>
(Office クライアント・アプリで AIP 保護されたファイルを開く際に Microsoft Account は利用可能だが、<br>
AIP Viewer、Edge、Acrobat などでは、AIP 保護された PDF ファイル等を開く際に、Microsoft Account の利用はできない)  

## 主な考慮事項
1. 外部ユーザーの指定方法
2. 外部ユーザー環境
3. B2B ワンタイム パスコードを利用した外部ユーザーの認証法
4. 条件付きアクセスの考慮事項
5. AIP 保護されたファイルを開く際のネットワーク要件

# 1. 外部ユーザーの指定方法
基本的に、Azure Information Protection では、メール アドレスでユーザーやグループを指定します。
またファイルを開く際には、Azure Active Dreictory で認識されている、ProxyAddresses に設定された単一もしくは複数のメール アドレスか、
ProxyAddresses に設定がなければ、UserPrincipalName の値で、指定されたメール アドレスに合致するかどうかを判定します。  
[参考: Preparing users and groups for Azure Information Protection](https://docs.microsoft.com/en-us/azure/information-protection/prepare)  

同様に外部ユーザーも、メール アドレスで指定します。なおメール アドレスが有効なグループ(セキュリティ、配布グループ、Microsoft 365 グループ)も利用することができ、自テナントのユーザーをグループに追加することや、Azure AD B2B の機能で外部ユーザーを招待していれば、そのゲスト アカウントを自テナントのグループに追加して権限を付与することも可能です。またメール アドレスが分かっていれば社外の Office 365 テナントのグループであっても、メール アドレスを指定し、権限制御に利用することが可能です。  
  
上記のような直接のユーザーやグループの指定以外にも、ドメイン単位で権限を設定することも可能ですが、設定できる方法が限られていて、エンドユーザーにおいては、AIP Clientをインストールし、エクスプローラー上で、ファイルを右クリックして「分類して保護する」のメニューから権限設定をする必要がある点には注意が必要です。なおドメインで許可した場合、それが Office 365 テナントで利用されているドメインであれば、そのドメインが利用されているテナント全体のユーザーに権限が与えられます。  
[参考: アクセス許可を今すぐ割り当てる](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/encryption-sensitivity-labels?view=o365-worldwide#assign-permissions-now)

## 外部ユーザーの指定方法まとめ
| 環境 | 個別のユーザー指定<br>(メール アドレスが有効な AAD のグループ指定含む) | ドメイン指定<br>(Azure AD ではそのドメインを持つテナント全体) | 認証されたユーザーは誰でも |
|:---:|:---:|:---:|:---:|
|Office 365 Apps for Business | 〇 | 〇 <br> (Microsoft 365 Apps の 2303 でネイティブに対応) | × |
|Azure Information Clienet をインストールした上で、ファイルを右クリックして権限設定する場合 | 〇 | 〇 | × |
|テナント管理者が秘密度ラベルの保護設定を定義する場合| 〇 | 〇 | 〇 |

## Microsoft 365 Apps -> ドメインの設定も Microsoft 365 Apps の 2303 でネイティブに対応
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_PIC01.png">  
(以前のバージョンでも、特殊な AllStaff-7184AB3F-CCD1-46F3-8233-3E09E9CF0E66@ドメイン名という指定方法でドメインのすべてのユーザーを指定することは可能。)  
  
## Azure Information Clienet をインストールした上で、ファイルを右クリックして権限設定する場合 -> ドメインの設定ができる  
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_PIC02.png">
   
## テナント管理者が秘密度ラベルの保護設定を定義する場合 -> ドメインの設定ができる  
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_PIC03.png">
   
なおファイルの暗号化はするものの、認証されたユーザーは誰でも開いてよいという場合、テナント管理者が秘密度ラベルの保護設定を定義する際に、「任意の認証済みユーザー」を指定することもできます。この場合、誰がファイルを開けるかは一切コントロールしないため、有効期限の設定をしてファイルの参照期間を限定することや、追跡・失効の機能を併用した使い方が考えられます。

# 2. 外部ユーザー環境
AIP 保護されたファイルを開く際には、AIP に対応した Office 環境等が必要となり、環境やアプリにより利用可能なアカウントの種類が異なる点には注意が必要です。   
[参考: 保護されたドキュメントを開く場合にサポートされるシナリオ](https://docs.microsoft.com/ja-jp/azure/information-protection/secure-collaboration-documents#supported-scenarios-for-opening-protected-documents)

## 保護された Office ファイルを開く場合
| 環境 | Office 365 アカウントでの AIP 保護されたファイルの参照 | Microsoft Account での AIP 保護されたファイルの参照 | Azure AD B2B (ワンタイム パスコード含む)を利用した AIP 保護されたファイルの参照 |
|:---:|:---:|:---:|:---:|
|Microsoft 365 Apps | 〇 | 〇 | 〇 |
|Office 2019 | 〇 | 〇 | 〇 |
|Office 2013, 2016 | 〇 | × | 〇 |
|Office 2010 | 〇 開けるが AIP Client のインストール必須 | × | × 先進認証に対応しないため Azure AD B2B での利用は不可 |  
|iOS 版 Office (バージョン 2.42 以降) | 〇 | 〇 | 〇 |  
|Android 版 Office (バージョン 16.0.13029 以降) | 〇 | 〇 | 〇 |  
|MacOS 版 Office (バージョン 16.42 以降) | 〇 | 〇 | 〇 |  

## 保護された PDF ファイルを開く場合
| 環境 | Office 365 アカウントでの AIP 保護されたファイルの参照 | Microsoft Account での AIP 保護されたファイルの参照 | Azure AD B2B (ワンタイム パスコード含む)を利用した AIP 保護されたファイルの参照 |
|:---:|:---:|:---:|:---:|
|Windows/iOS/Android 版 AIP Viewer (その他の ptxt, pjpg などのファイル形式にも対応) | 〇 | × | 〇 |  
|Windows 8, 8.1, 10, 11 版 Edge（バージョン 83.0.478.37 以降) | 〇（PDFのみが対象) | × | 〇（PDFのみが対象)|  
|Windows 8, 8.1, 10, 11 版Acrobat / Acrobat Reader + [MIP plug-in for Acrobat and Acrobat Reader](https://go.microsoft.com/fwlink/?linkid=2050049) | 〇（PDFのみが対象) | × | 〇（PDFのみが対象) |  
|macOS 10.12 以上での Acrobat / Acrobat Reader + [MIP plug-in for Acrobat and Acrobat Reader](https://go.microsoft.com/fwlink/?linkid=2050049) | 〇（PDFのみが対象) | × | 〇（PDFのみが対象) |  

AIP 保護された PDF ファイルの参照  
[参考: 保護された PDF では、どの PDF リーダーがサポートされていますか?](https://docs.microsoft.com/ja-jp/azure/information-protection/rms-client/protected-pdf-readers)  
[参考: 外部ユーザーと AIP ビューアー](https://docs.microsoft.com/ja-jp/azure/information-protection/known-issues#known-issues-for-the-aip-viewer)   
なお Windows 版の Acrobat Reader + MIP Plug-in の場合、既存認証情報は、 %userprofile%\AppData\Roaming\Adobe\Acrobat\DC\Security に格納されているため、ユーザーを切り替えたり、認証をやり直したい場合には、自己責任でこちらのフォルダのファイルを適宜削除のこと。

# 3. B2B ワンタイム パスコードを利用した外部ユーザーの認証法
AIP 保護されたファイルの共有先が、Exchange Online ではないメール サーバーを利用していたり、コンシューマー向けの各種インターネット サービス プロバイダーで提供されているメール アカウントを利用している場合には、AIP で保護されたファイルを開く際のアカウントを持っていません。この場合、Teams のチームへの招待や、SharePoint Online の外部共有機能を用いて、汎用的なワンタイム パスコード ベースの Azure AD B2B 招待をして対応することができます。(他にも、[Azure Information Protection](https://aka.ms/rms-signup) サイトにて、Azure Active Directory の Free のアカウントを作成してもらうことも可能ですが、コンシューマー向けのメール ドメインを持つメール アドレスの利用は不可となります。) ワンタイム パスコード ベースの Azure AD B2B 招待を利用する場合、テナントで以下のような前提条件を満たしている必要があります。

## ワンタイム パスコード ベースの Azure AD B2B 招待の前提条件
1. Azure AD の B2B で[ワンタイム パスコード](https://docs.microsoft.com/ja-jp/azure/active-directory/external-identities/one-time-passcode)が有効化されていること  
(オプト アウトしていない場合、2021 年 11 月 1 日以降より順次有効化)  
2. SharePoint Online / OneDriver for Business で外部招待をする場合には、[B2B 統合機能](https://docs.microsoft.com/ja-jp/sharepoint/sharepoint-azureb2b-integration)が有効されていること
3. 権限があるユーザーで B2B の外部招待を一度実施していること   
(テナントで外部招待が制限されている場合、Guest Inviter のロール等で、招待権限がある人の操作により、対象の外部ユーザーが B2B 招待されていること
    
上記のワンタイム パスコード ベースの Azure AD B2B 招待ができていれば、社外のユーザーのメール アドレスやドメインを指定して AIP 保護したファイルを用意し送付することで、社外のユーザーにも、AIP 保護したファイルを開いてもらうことができます。なお B2B の仕組みを用いて認証する場合には、共有元となるテナントのドメイン情報が必要となるので、**AIP 保護されたファイルを開いてもらう際に利用する、共有元テナントのドメイン情報も併せて伝えておく必要があります**。

## 共有先での B2B/ワンタイム パスコードを利用した認証方法
1. 保護されたファイルを開く際、職場または学校アカウントを選択し、自身のメール アドレスを一度入力する
2. サインイン画面で、必ず「サインイン オプション」を選択し、「**組織へのサインインを選択する**」<br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP1.png"> <br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP2.png"> <br>
3. 組織の検索で、**ファイル共有元から提示してもらったドメイン名を入力する**<br>(通常は共有元のメール アドレスのドメインで OK) <br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP3.png"> <br>
5. 続くダイアログで、共有元のテナント環境のブランドが適用された認証画面が表示されることを確認する
6. 自身のメール アドレスが入力されていることを確認し、次へをクリック<br>(一度認証済みであれば、自身のメール アドレスにコードを送信する画面となるので「コード」の送信をクリック <br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP4.png"> <br>
8. 自身のメール アドレスに 8 桁のワンタイム パスコードが届くので、30 分以内にそのコードを、認証画面に入力して、認証を完了 <br>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_OTP5.png"> <br>
9. 共有された AIP 保護ファイルで権限設定されたメール アドレスやドメインに合致していれば、その権限設定の範囲で、ファイルの参照や利用が可能

なお SharePoint Online を通じて [B2B 統合機能](https://docs.microsoft.com/ja-jp/sharepoint/sharepoint-azureb2b-integration)が有効された状態で外部招待するのであれば、SharePoint Online のライブラリで、[IRM](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/apply-irm-to-a-list-or-library?view=o365-worldwide) 機能を有効化し、ライブラリから対応したファイルをダウンロードする際には、自動的に AIP で保護するという構成も取ることができます。

# 4. 条件付きアクセスの考慮事項
AIP のファイルを開く際に、共有元テナントでの条件付きアクセスによる制御を実施することが可能ですが、すべてのユーザーに対して、AIP の条件付きアクセス制御を設定する場合は注意が必要です。通常、共有先が Office 365 のユーザーであり、各自のテナントで Azure Active Directory のアカウントを持っている場合、Azure Activit Directory のテナント間の暗黙的な信頼により、特に明示的な B2B の招待がなくても、AIP ファイルを開いてもらうことが可能です。ただし、**すべてのユーザーに対してAIP の条件付きアクセス制御を設定した場合**、外部のユーザーも例外ではなく、この条件付きアクセス制御の対象となりますが、認識すべき制約として、**自社テナント側で該当する社外ユーザーの B2B アカウントが存在していない場合**、社外ユーザーによる AIP へのアクセスがすべてブロックされ、認証を通すことができなくなり、結果 **B2B アカウントが存在していない社外ユーザーは、 AIP で保護されたファイルを開けなくなります**。そのため、外部ユーザー含めて AIP の条件付きアクセス制御を展開したい場合には、相手が Office 365 のユーザーや、Microsoft アカウントの利用者であっても、必ず B2B 招待するようにしておく必要があります。一方で B2B 招待の必要性を無くしたい場合には、すべてのユーザーに対する条件付きアクセス制御のポリシーから、AIP を除外することや、個別に AIP の条件付きアクセスを設定したい場合には、特定の B2B 招待されたユーザーやそれらユーザーを含むグループのみを対象とした条件付きアクセス ポリシーを設定するようにします。   
[参考: AIP ベースの条件付きアクセス ポリシー](https://docs.microsoft.com/ja-jp/azure/information-protection/known-issues#aip-based-conditional-access-policies)

# 5. AIP 保護されたファイルを開く際のネットワーク要件
AIP 保護されたファイルを開く際、クラウドへの接続および認証が発生します。そのため、以下のような URL への接続が許可されている必要があります。
- login.microsoftonline.com
- \*.aadrm.com 
- \*.azurerms.com
- \*.informationprotection.azure.com
- \*.cloudapp.net
- ecn.dev.virtualearth.net
- informationprotection.hosting.portal.azure.net
- \*.protection.outlook.com   
- config.edge.skype.com
- \*.events.data.microsoft.com
- \*.aria.microsoft.com (Android デバイスのみ)   

また以下のプロキシの要件にも注意する必要があります。
- 認証付きプロキシを利用していないこともしくは認証が必要なプロキシを使っている場合であっても、統合 Windows 認証が利用でき、デフォルトの認証情報を利用するよう AIP のレジストリ設定をしていること
- aadrm.com との通信で、HTTPS インスペクションをしていないこと (既定の Micorsoft CA の証明書を置き換えた HTTPS 通信になっていないこと)

詳細については、以下を確認ください。    
[参考: Azure Information Protection の要件](https://docs.microsoft.com/ja-jp/azure/information-protection/requirements#firewalls-and-network-infrastructure)


