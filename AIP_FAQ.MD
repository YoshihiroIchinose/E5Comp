# Azure Informatino Protection の外部共有について (書きかけ)
Azure Informatino Protection では、自社テナントだけではなく、以下のような社外アカウントを利用するユーザーとも暗号化されたファイルの共有が可能となっています。  
A. 外部の Office 365 テナントのアカウント
B. Microsoft Account を利用するコンシューマー アカウント
C. Azure AD B2B の仕組みを使ったメール アドレス＋ワンタイプ パスコードを利用する外部ユーザー アカウント

ただし、ファイルを共有しようとするユーザーからは完全にコントロールすることが難しい、
各種前提条件もあり、あまり活用が進んでいないかと思われます。本ページでは、
利用可否の判断をより具体的に行ってもらうことを目的に、これら前提条件や注意事項を明確にします。
主な考慮事項
1. 外部ユーザーの指定方法
2. 外部ユーザーの Office 環境
3. 外部ユーザーの認証法
4. 条件付きアクセスの考慮事項

# 1. 外部ユーザーの指定方法
基本的に、Azure Information Protection では、メール アドレスでユーザーやグループを指定します。
またファイルを開く際には、Azure Active Dreictory で認識されている、ProxyAddresses に設定された単一もしくは複数のメール アドレスか、
ProxyAddresses に設定がなければ、UserPrincipalName の値で、指定されたメール アドレスに合致するかどうかを判定します。
[参考: Preparing users and groups for Azure Information Protection](https://docs.microsoft.com/en-us/azure/information-protection/prepare)
同様に外部ユーザーも、メール アドレスで指定します。
メール アドレスが有効なグループ(セキュリティ、配布グループ、Microsoft 365 グループ)
