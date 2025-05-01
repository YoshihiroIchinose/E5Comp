# Graph API を利用して SharePoint Online のファイルに秘密度ラベルを付与する
SharePoint Online のファイルに Graph API を利用して秘密度ラベルを付与するサンプルです。
Graph API を用いた秘密度ラベルの付与では、1 操作につき、[$0.0018 の従量課金](https://learn.microsoft.com/ja-jp/graph/metered-api-list)が発生するため、Azure のサブスクリプション環境が必要となります。


## 事前準備

### アプリケーションの登録
1. Azure AD (https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps) でアプリケーションを登録する
2. 作成したアプリケーションの概要ページから、アプリケーション (クライアント) ID の値をコピーしておく。
3. 管理の API のアクセス許可で、アプリケーションの許可として Sites.ReadWrite.All の権限を与える
4. 付与した権限に対して管理者の同意を与える
5. 作成したアプリケーションの Client ID を用いて以下の URL の後ろのパラーメーターを書き換え   
6. 管理の証明書とシークレットから Client Secret を新規作成して値をコピーしておく

### 従量課金の API の有効化
