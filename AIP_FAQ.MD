# Azure Informatino Protection の外部共有について (書きかけ)
Azure Informatino Protection では、自社テナントだけではなく、以下の方式により、社外のユーザーとも暗号化されたファイルの共有が可能となっています。
1. Azure Active Dreictory  間の暗黙の信頼関係による外部の Office 365 テナントのユーザー
2. Microsoft Account
3. Azure AD B2B の仕組みも併せて利用することで、メール アドレスのみの外部のユーザーにも利用可能となっています

ただし、ファイルを共有しようとするユーザーからは完全にコントロールすることが難しい、
各種前提条件もあり、あまり活用が進んでいないかと思われます。本ページでは、
利用可否の判断をより具体的に行ってもらうことを目的に、これら前提条件や注意事項を明確にします。

# 外部ユーザーへの権限の設定方法
基本的に、Azure Information Protection では、メール アドレスでユーザーやグループを指定します。
またファイルを開く際には、Azure Active Dreictory で認識されている、ProxyAddresses に設定された単一もしくは複数のメール アドレスか、
ProxyAddresses に設定がなければ、UserPrincipalName の値で、指定されたメール アドレスに合致するかどうかを判定します。
[参考: Preparing users and groups for Azure Information Protection](https://docs.microsoft.com/en-us/azure/information-protection/prepare)
同様に外部ユーザーも、メール アドレスで指定します。
メール アドレスが有効なグループ(セキュリティ、配布グループ、Microsoft 365 グループ)
