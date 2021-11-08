# カスタムのSIT定義で正規表現のエラーが出た場合
Office 365 のData Classfication Service では、現在 C++ の Boost.Regex 5.1.3 のエンジンが利用されています。  
[参考： カスタムの機密情報の種類を使用する前に - Microsoft 365 Compliance | Microsoft Docs](https://docs.microsoft.com/ja-jp/microsoft-365/compliance/create-a-custom-sensitive-information-type?view=o365-worldwide)  
SIT の正規表現についてですが、手動で登録した SIT は内部的にすべて共通のルール パッケージにまとめられおり、他のどれかの正規表現に仕様変更などでエラーが発生すると、エラーがない SIT の編集もすべてエラーとなってしまいます。

