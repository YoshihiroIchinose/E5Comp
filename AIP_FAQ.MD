# Azure Informatino Protection で保護されたファイルを外部ユーザーと共有する (書きかけ)
Azure Informatino Protection では、自社テナントだけではなく、以下のような社外アカウントを利用するユーザーとも暗号化されたファイルの共有が可能となっています。  
A. 外部の Office 365 テナントのアカウント  
B. [Azure Information Protection](https://aka.ms/rms-signup) にて会社のメール アドレスを用いて個別にサインアップしたアカウント  
C. コンシューマーの Microsoft Account 
D. Azure AD B2B の仕組みを使ったメール アドレス＋ワンタイプ パスコードを利用する外部ユーザー アカウント  

ただし、ファイルを共有しようとするユーザーからは完全にコントロールすることが難しい、
各種前提条件もあり、あまり活用が進んでいないかと思われます。本ページでは、
利用可否の判断をより具体的に行ってもらうことを目的に、これら前提条件や注意事項を明確にします。
主な考慮事項
1. 外部ユーザーの指定方法
2. 外部ユーザー環境
3. 外部ユーザーの認証法
4. 条件付きアクセスの考慮事項

# 1. 外部ユーザーの指定方法
基本的に、Azure Information Protection では、メール アドレスでユーザーやグループを指定します。
またファイルを開く際には、Azure Active Dreictory で認識されている、ProxyAddresses に設定された単一もしくは複数のメール アドレスか、
ProxyAddresses に設定がなければ、UserPrincipalName の値で、指定されたメール アドレスに合致するかどうかを判定します。  
[参考: Preparing users and groups for Azure Information Protection](https://docs.microsoft.com/en-us/azure/information-protection/prepare)  

同様に外部ユーザーも、メール アドレスで指定します。なおメール アドレスが有効なグループ(セキュリティ、配布グループ、Microsoft 365 グループ)も利用することが可能です。
またメール アドレスが分かっていれば社外の Office 365 テナントのグループであっても、メール アドレスを指定し、権限制御に利用することが可能です。  
  
上記のような直接のユーザーやグループの指定以外にも、ドメイン単位で権限を設定することも可能ですが、設定できる方法が限られていて、エンドユーザーにおいては、AIP Clientをインストールし、エクスプローラー上で、ファイルを右クリックして「分類して保護する」のメニューから権限設定をする必要がある点には注意が必要です。なおドメインで許可した場合、それが Office 365 テナントで利用されているドメインであれば、そのドメインが利用されているテナント全体のユーザーに権限が与えられます。  
[参考: アクセス許可を今すぐ割り当てる](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/encryption-sensitivity-labels?view=o365-worldwide#assign-permissions-now)

- Office クライアント -> ドメインの設定ができない
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_PIC01.png">  
(特殊な AllStaff-7184AB3F-CCD1-46F3-8233-3E09E9CF0E66@ドメイン名という指定方法でドメインのすべてのユーザーを指定することは可能。)  
  
- Azure Informatino Clienet をインストールした上で、ファイルを右クリックして権限設定する場合 -> ドメインの設定ができる  
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_PIC02.png">
- テナント管理者が秘密度ラベルの保護設定を定義する際 -> ドメインの設定ができる  
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/AIP_PIC03.png">

なおファイルの暗号化はするものの、認証されたユーザーは誰でも開いてよいという場合、テナント管理者が秘密度ラベルの保護設定を定義する際に、「任意の認証済みユーザー」を指定することもできます。
この場合、誰がファイルを開けるかは一切コントロールしないため、有効期限の設定をしてファイルの参照期間を限定することや、追跡・失効の機能を併用した使い方が考えられます。

# 2. 外部ユーザー環境
AIP 保護された Office ファイルの参照  
[参考: 保護されたドキュメントを開く場合にサポートされるシナリオ](https://docs.microsoft.com/ja-jp/azure/information-protection/secure-collaboration-documents#supported-scenarios-for-opening-protected-documents)
| 環境 | Office 365 アカウントでの AIP 保護されたファイルの参照 | Microsoft Account での AIP 保護されたファイルの参照 | Azure AD B2B (ワンタイム パスコード含む)を利用した AIP 保護されたファイルの参照 |
|:---:|:---:|:---:|:---:|
|Office 365 Apps for Business | 〇 | 〇 | 〇 |
|Office 2019 | 〇 | 〇 | 〇 |
|Office 2013, 2016 | 〇 | × | 〇 |
|Office 2010 | 〇 開けるが AIP Client のインストール必須 | × | × 先進認証に対応しないため Azure AD B2B での利用は不可 |  

AIP 保護された PDF ファイルの参照  
[参考: 保護された PDF では、どの PDF リーダーがサポートされていますか?](https://docs.microsoft.com/ja-jp/azure/information-protection/rms-client/protected-pdf-readers)
- Windows 8, 8.1, 10, 11 上の Edge バージョン 83.0.478.37 以降
- Windows 8, 8.1, 10, 11 上の [AIP Client](https://aka.ms/aipclient) もしくは [AIP Viewer](https://go.microsoft.com/fwlink/?linkid=838993)
- Windows 8, 8.1, 10, 11 上の Acrobat / Acrobat Reader + [MIP plug-in for Acrobat and Acrobat Reader](https://go.microsoft.com/fwlink/?linkid=2050049)
- macOS 10.12 以上での Edge バージョン 83.0.478.37 以降
- macOS 10.12 以上での Acrobat / Acrobat Reader + [MIP plug-in for Acrobat and Acrobat Reader](https://go.microsoft.com/fwlink/?linkid=2050049)
- iOS/Android 用 Azure Information Protection アプリ  

# 3. 外部ユーザーの認証法
# 4. 条件付きアクセスの考慮事項
