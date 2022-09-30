# Microsoft Purview (E5 Compliance) 研究所
Microsoft Purview (E5 Compliance) に関する技術検証結果などを共有するサイトです。

| ページ | 内容 |
| --- | ---- |
| [日本向け個人情報の定義](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SIT.md) | 住所、電話番号など、M365 での利用を想定した日本向けの正規表現のパターンの例についてです。 |
| [カスタムの機密情報の正規表現のエラーの対処](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SIT_RegEx.md) | カスタムの機密情報定義の正規表現で、エラーになった場合の対処方法についてです。 |
| [Communication Compalince 用語サンプル](https://github.com/YoshihiroIchinose/E5Comp/blob/main/CC.md) | コミュニケーション コンプライアンスのテスト用に定義したサンプルの要注意ワードです。 |
| [Office 365 監査ログに関する技術情報](https://github.com/YoshihiroIchinose/E5Comp/blob/main/Office365Audit.md) | Office 365 Audit ログを最大 5 万件の範囲で CSV 形式で取得するサンプル スクリプトです。また AuditData のパースや、CSV ファイルの Excel ファイルへの変換などにも触れています。 |
| [Office 365 の監査ログの RecordTypes と保持](https://github.com/YoshihiroIchinose/E5Comp/blob/main/RecordTypes.md) | Office 365 Aduit ログに記録されるログの種類と、それらの種類を指定したログの保持ポリシーの設定方法について紹介しています。 |
|[Azure Automation で Office 365 の監査ログから特定の操作を CSV 形式で出力し SharePoint Online サイトにアップロードする](https://github.com/YoshihiroIchinose/E5Comp/blob/main/UploadAuditLogtoSPO.md) | Azure Automation を利用して Office 365 の監査ログから特定の操作を CSV 形式で出力し SharePoint Online サイトにアップロードするスクリプトです。定期的に SharePoint Online に CSV 形式でログを出力しておくことで、Power BI のレポートなどにも取り込めます。|
| [監視対象のユーザーの許可されていない Teams チームへの参加](https://github.com/YoshihiroIchinose/E5Comp/blob/main/TeamsMembership.MD)| ネストは考慮せず特定のグループに属する監視対象のユーザーが、許可されていない Teams への参加があれば、それらを出力するサンプル スクリプトを紹介しています。 |
| [利益相反するグループで共通する Teams チームを見つける](https://github.com/YoshihiroIchinose/E5Comp/blob/main/ConflictsOfInterestInTeams.md) | 利益相反するグループ間のコミュニケーションを監視する方法の一つとして、ネストは考慮せず特定のGroupAに属するメンバーとGroup Bに属するメンバーが、許可されていない Teams チームで、メンバーとして共存しているのであれば、それらを列挙するサンプル スクリプトを紹介しています。 |
| [各サービスにおける端末上でのファイル操作ログの出方の違い](https://github.com/YoshihiroIchinose/E5Comp/blob/main/MDE_DLP_IRM.MD) | Defender for Endpoint, Endpoint DLP, Insider Risk Managment のそれぞれでどういった観点のログが記録されるかについて。 |
|[Azure Information Protection で保護されたファイルを外部ユーザーと共有する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AIP_FAQ.MD)| AIP 保護されたファイルを社外の外部ユーザーと共有する際の要件や注意事項についてまとめています。|
| [社外とのセキュアなファイル共有方法](https://github.com/YoshihiroIchinose/E5Comp/blob/main/ExternalSharing.md) | PPAP に代わり社外のユーザーとセキュアにファイル共有する方法についてまとめました。 |
| [サービス プランの割り当て状況に応じたセキュリティ グループのメンテナンス](https://github.com/YoshihiroIchinose/E5Comp/blob/main/LicensePowerShell.md) | ライセンスを細分化したアプリ単位となるサービス プランの割り当て状況に応じたセキュリティ グループのメンテナンスを実施する PowerShel スクリプトのサンプルです。 |
| [Office 365 管理用の PowerShell スクリプトでの資格情報の管理手法について](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SecretsInScripts.md) | 各種 Office 365 管理用のスクリプトの中で、無人で定期実行したい場合には、対話的に資格情報を入力できないため資格情報を何等かの形で事前に持たせておく必要があります。 そういったスクリプトの用途に利用可能な資格の保管・呼び出し方法についてまとめています。 |
|[Office 365 Management API を利用した監査ログの取得について](https://github.com/YoshihiroIchinose/E5Comp/blob/main/O365MgmtAPI.md)|Search-UnifiedAuditLog よりも大規模に対応する Office 365 Management API を利用した PowerShell からの Office 365 監査ログの取得サンプルです。|
|[機密情報や Exact Data Match の動作に関して](https://github.com/YoshihiroIchinose/E5Comp/blob/main/EDMMatching.MD)| 機密情報の検出や、Exact Data Match における日本語環境での現状の動作をまとめています。 |
|[EDM のデータ アップロード用に全角英数字・記号を半角に変換する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/EDM_Preprocess.md)|EDM のデータ アップロード用に CSV ファイルの中の全角の英数字・記号を半角に変換する PowerShell スクリプトのサンプルです。|
|[Insider Risk Management の HR Connector に関連したスクリプト](https://github.com/YoshihiroIchinose/E5Comp/blob/main/IRM_HR_Connector.md)|Azure Storage にアップロードされた CSV ファイルを、Azure Automation のスクリプトを通じて IRM の HR Connector に取り込むサンプルと、HR Connector で取り込む CSV ファイルを元に、メールが有効なセキュリティ グループを作成し、グループメンバーシップをメンテナンスするサンプルを紹介しています。|
|[Graph API で Teams チャネルに投稿する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/PostTeamsMessage2.md)|CC / DLP のテストなどに利用するため、Teams のチャネル チャットへの投稿を自動化するスクリプトのサンプルです。|
|[アクティビティ エクスプローラのログの CSV への書き出しサンプル](https://github.com/YoshihiroIchinose/E5Comp/blob/main/ActivityExplorerData.md)|新たに追加された Export-ActivityExplorerData のコマンドを利用して、最大 30 日分、5 万行 (5,000 行x 10 回)のログを取得するサンプルです。|
