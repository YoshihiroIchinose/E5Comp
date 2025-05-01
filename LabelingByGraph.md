# Graph API を利用して SharePoint Online のファイルに秘密度ラベルを付与する
SharePoint Online のファイルに Graph API を利用して秘密度ラベルを付与するサンプルです。
Graph API を用いた秘密度ラベルの付与では、1 操作につき、[$0.0018 の従量課金](https://learn.microsoft.com/ja-jp/graph/metered-api-list)が発生するため、Azure のサブスクリプション環境が必要となります。


## 事前準備

### アプリケーションの登録
1. Azure AD (https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps) でアプリケーションを登録する
2. 作成したアプリケーションの API のアクセス許可で、委任されたアクセス許可として Team.ReadBasic.All、Channel.ReadBasic.All、ChannelMessage.Send の権限を与える    
(ChannelMessage.Send はアプリケーションとしては実行できず、必ずユーザーの委任としての操作が必要。）
3. 作成したアプリケーションの Client ID を用いて以下の URL の後ろのパラーメーターを書き換え   
   https://login.microsoftonline.com/common/oauth2/authorize?response_type=code&client_id=XXXXX   
その URL に InPrivate モードなどの状態のブラウザでアクセスし、利用するアカウントで認証を通し権限の同意を実施する   
認証が通り権限に同意できていれば「AADSTS500113: No reply address is registered for the application.」というエラーになるがこの状態になれば OK 
4. 事前準備したアプリケーションの Client Secret を新規作成する
5. 以下のスクリプトで、作成したアプリケーションの ClientID、Client Secret、ユーザーの ID/パスワード、および投稿先のチーム名、チャネル名、投稿内容を書き換える
