# Microsoft Purview (E5 Compliance) 研究所
Microsoft Purview (E5 Compliance) に関する技術検証結果などを共有するサイトです。

# Purview 全般
| ページ | 内容 |
| --- | ---- |
| [日本向け個人情報の定義](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SIT.md) | 住所、電話番号など、M365 での利用を想定した日本向けの正規表現のパターンの例についてです。 |
| [カスタムの機密情報の正規表現のエラーの対処](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SIT_RegEx.md) | カスタムの機密情報定義の正規表現で、エラーになった場合の対処方法についてです。 |
| [各サービスにおける端末上でのファイル操作ログの出方の違い](https://github.com/YoshihiroIchinose/E5Comp/blob/main/MDE_DLP_IRM.MD) | Defender for Endpoint, Endpoint DLP, Insider Risk Managment のそれぞれでどういった観点のログが記録されるかについて。 |
| [社外とのセキュアなファイル共有方法](https://github.com/YoshihiroIchinose/E5Comp/blob/main/ExternalSharing.md) | PPAP に代わり社外のユーザーとセキュアにファイル共有する方法についてまとめました。 |
|[SharePoint Online のライブラリ単位で DLP を適用・除外する方法](https://github.com/YoshihiroIchinose/E5Comp/blob/main/DLPforSPOLibrary.md)|ライブラリに対する既定の保持ラベルを DLP の条件とすることで、SharePoint Online のサイト内で、ライブラリ単位に DLP の適用や、DLP の適用除外を制御する方法について紹介します。|
|[Microsoft Sentinel (Log Analytics) の特定のログを削除する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SentinelDataPurge.md)|Azure Automation を利用して、 Log Anlytics から AIPHeartBeat のログのみを削除するサンプル コードを共有します。|

# 秘密度ラベル関連記事
| ページ | 内容 |
| --- | ---- |
|[Azure Information Protection で保護されたファイルを外部ユーザーと共有する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AIP_FAQ.MD)| AIP 保護されたファイルを社外の外部ユーザーと共有する際の要件や注意事項についてまとめています。|
|[Sentinel で秘密度ラベルのダウングレード操作をアラートする](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SentinelKusto.md)|Microsoft Purview Information Protection のデータ コネクタで取り込んだ秘密度ラベルの操作のログを元に、 一定期間内に複数回秘密度ラベルのダウングレードがあった場合に、それらをアラートする仕組みを Kusto クエリで作成するサンプルです。|
|[アラート ポリシーを利用して監査ログに記録された操作に応じてアラートを発令する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AlertPolicy.md)|アラート ポリシーを、PowerShell から設定することで、1 時間の中で 3 回以上、秘密度ラベルを変更するとアラートを生成する方法について紹介します。|
|[秘密度ラベルの操作をデスクトップ上のドラッグ アンド ドロップの操作で行う](https://github.com/YoshihiroIchinose/E5Comp/blob/main/LabelbyDD.md)|Purview Information Protection Client に含まれる ラベル付けおよびラベル削除のPowerShell コマンドレットを用いて、 ドラッグ アンド ドロップの操作で、複数のファイルに対して秘密度ラベル付けや、ラベルの削除を行うサンプルです。|
|[Graph API を利用して SharePoint Online のファイルに秘密度ラベルを付与する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/LabelingByGraph.md)|SharePoint Online のファイルに Graph API を利用して秘密度ラベルを付与するサンプルです。|

# 監査ログ関連記事
| ページ | 内容 |
| --- | ---- |
| [Office 365 監査ログに関する技術情報](https://github.com/YoshihiroIchinose/E5Comp/blob/main/Office365Audit.md) | Office 365 Audit ログを最大 5 万件の範囲で CSV 形式で取得するサンプル スクリプトです。また AuditData のパースや、CSV ファイルの Excel ファイルへの変換などにも触れています。 |
| [Office 365 の監査ログの RecordTypes と保持](https://github.com/YoshihiroIchinose/E5Comp/blob/main/RecordTypes.md) | Office 365 Aduit ログに記録されるログの種類と、それらの種類を指定したログの保持ポリシーの設定方法について紹介しています。 |
|[Office 365 Management API を利用した監査ログの取得について](https://github.com/YoshihiroIchinose/E5Comp/blob/main/O365MgmtAPI.md)|Search-UnifiedAuditLog よりも大規模に対応する Office 365 Management API を利用した PowerShell からの Office 365 監査ログの取得サンプルです。|
|[Azure Automation で Office 365 の監査ログから特定の操作を CSV 形式で出力し SharePoint Online サイトにアップロードする](https://github.com/YoshihiroIchinose/E5Comp/blob/main/UploadAuditLogtoSPO.md) | Azure Automation を利用して Office 365 の監査ログから特定の操作を CSV 形式で出力し SharePoint Online サイトにアップロードするスクリプトです。定期的に SharePoint Online に CSV 形式でログを出力しておくことで、Power BI のレポートなどにも取り込めます。|
|[B2B コラボレーションを中心とした外部共有のログの確認](https://github.com/YoshihiroIchinose/E5Comp/blob/main/ExternalSharingLogs.md)|B2B コラボレーションで、外部共有先を把握するために、3 つの操作を対象に共有先のゲスト ユーザーを含め外部共有に関するログを CSV 形式で抽出するスクリプトです。|
|[許可されていないドメインへの外部共有をメール通知する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/ExternalSharingMonitoring.md)|SharePoint リストで用意する許可されたドメイン一覧の情報も加味しながら、許可されたドメイン以外の外部ユーザーへの共有をログから見つけて、Azure Automate/Power Automate/SharePoint Online リストを駆使して、共有操作を行ったユーザーに注意喚起のメール通知を行うソシューション サンプルです。|
|[秘密度ラベル操作をメール通知する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/LabelMonitoring.md)|M365 Apps、SharePoint (Office for the web)、AIP アドインのそれぞれの監査ログから、 秘密ラベルのダウングレード・秘密度ラベルの削除・サブ ラベルの変更による暗号化解除の操作を抜き出し、これら操作のログを、 Azure Automate を利用してSharePoint リストに書き込むと共に、Power Automate を利用して、カスタムのメール通知を行うソリューションの実装サンプルです。|
| [Endpoint DLP の操作をメール通知する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/EDLPAlert.md) | 監査ログから Endpoint DLP で監視される、FileCopiedToRemovableMedia および FileUploadedToCloud の操作を抜き出し、これら操作のログを、 Azure Automate を利用してSharePoint リストに書き込むと共に、Power Automate を利用して、カスタムのメール通知を行うソリューションの実装サンプルです。 |

# 運用スクリプト全般
| ページ | 内容 |
| --- | ---- |
| [監視対象のユーザーの許可されていない Teams チームへの参加](https://github.com/YoshihiroIchinose/E5Comp/blob/main/TeamsMembership.MD)| ネストは考慮せず特定のグループに属する監視対象のユーザーが、許可されていない Teams への参加があれば、それらを出力するサンプル スクリプトを紹介しています。 |
| [利益相反するグループで共通する Teams チームを見つける](https://github.com/YoshihiroIchinose/E5Comp/blob/main/ConflictsOfInterestInTeams.md) | 利益相反するグループ間のコミュニケーションを監視する方法の一つとして、ネストは考慮せず特定のGroupAに属するメンバーとGroup Bに属するメンバーが、許可されていない Teams チームで、メンバーとして共存しているのであれば、それらを列挙するサンプル スクリプトを紹介しています。 |
| [SharePoint Online および OneDrive for Business の各サイトに設定されたゲスト ユーザーをリストする (SPO Shell)](https://github.com/YoshihiroIchinose/E5Comp/blob/main/ExternalUsersFromSPO.md) | Azure Automation から、SharePoint Online Management Shell を利用して、 SharePoint Online および　OneDrive for Business の各サイトでのゲスト ユーザー一覧を取得するサンプル スクリプトです。なお取得したゲスト ユーザー一覧の情報は、CSV ファイルとして、特定の SharePoint Online サイトのドキュメント ライブラリに格納します。 |
| [サービス プランの割り当て状況に応じたセキュリティ グループのメンテナンス](https://github.com/YoshihiroIchinose/E5Comp/blob/main/LicensePowerShell.md) | ライセンスを細分化したアプリ単位となるサービス プランの割り当て状況に応じたセキュリティ グループのメンテナンスを実施する PowerShel スクリプトのサンプルです。 |
| [Office 365 管理用の PowerShell スクリプトでの資格情報の管理手法について](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SecretsInScripts.md) | 各種 Office 365 管理用のスクリプトの中で、無人で定期実行したい場合には、対話的に資格情報を入力できないため資格情報を何等かの形で事前に持たせておく必要があります。 そういったスクリプトの用途に利用可能な資格の保管・呼び出し方法についてまとめています。 |
|[Insider Risk Management の HR Connector に関連したスクリプト](https://github.com/YoshihiroIchinose/E5Comp/blob/main/IRM_HR_Connector.md)|Azure Storage にアップロードされた CSV ファイルを、Azure Automation のスクリプトを通じて IRM の HR Connector に取り込むサンプルと、HR Connector で取り込む CSV ファイルを元に、メールが有効なセキュリティ グループを作成し、グループメンバーシップをメンテナンスするサンプルを紹介しています。|
|[Graph API で Teams チャネルに投稿する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/PostTeamsMessage2.md)|CC / DLP のテストなどに利用可能な Graph API を PowerShell から利用して実装する Teams のチャネル チャットへの投稿を自動化するスクリプトのサンプルです。|
|[Microsoft Graph PowerShell SDK を通じて Teams チャネルに投稿する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/PostTeamsMessage3.md)|CC / DLP のテストなどに利用可能な Microsoft Graph PowerShell SDK を通じた Teams のチャネル チャットへの投稿を自動化するスクリプトのサンプルです。|
|[アクティビティ エクスプローラのログの CSV への書き出しサンプル](https://github.com/YoshihiroIchinose/E5Comp/blob/main/ActivityExplorerData.md)|新たに追加された Export-ActivityExplorerData のコマンドを利用して、最大 30 日分、5 万行 (5,000 行x 10 回)のログを取得するサンプルです。|
|[Graph API を用いて eDiscovery (Premium) のメールボックス検索を行う](https://github.com/YoshihiroIchinose/E5Comp/blob/main/eDiscoveryByGraph.md)|Graph API を用いて eDiscvoery(Premium) のケース作成、カストーディアン(調査対象となるユーザー)の追加、 検索対象としてカストーディアンのメールボックスの指定、メールからクレジット カード番号の SIT を見つける検索キーワードの登録、 レビュー セットの作成、レビューセットへの検索結果の反映までを自動化する例を示します。|

# Defender for Cloud Apps 関連記事
| ページ | 内容 |
| --- | ---- |
| [SharePoint Online および OneDrive for Business の各サイトに設定されたゲスト ユーザーをリストする(MDA)](https://github.com/YoshihiroIchinose/E5Comp/blob/main/MDA_API_External.md) | Azure Automation から、Defender for Cloud Apps の REST API をコールし、SharePoint Online および　OneDrive for Business にて、外部に共有されたファイルを取得し、その権限設定から、各サイトのゲスト ユーザー一覧を取得するサンプル スクリプトです。なお取得したゲスト ユーザー一覧の情報は、CSV ファイルとして、特定の SharePoint Online サイトのドキュメント ライブラリに格納します。 |
| [Defender for Cloud Apps のファイル ページの情報を API 経由で取得する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/MDA_API.md) 　| Microsoft 365 Defender のファイル ページの情報を、PowerShell を使った API アクセスで取得する方法について紹介します。|
| [Defender for Cloud Apps でのラベル適用のガバナンス アクションをリトライする](https://github.com/YoshihiroIchinose/E5Comp/blob/main/MDARetryLabeling.md) | Microsoft 365 Defender のガバナンス ログの情報を、 PowerShell を使った API アクセスで取得し、失敗したままになっている秘密度ラベル適用をリトライするものです。 |
| [Defender for Cloud Apps の API を利用して特定フォルダ内のファイルに秘密度ラベルを付与する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/MDALabeltoNonLabeldFiles.md) | Defender for Cloud Apps の API を利用して、特定フォルダ内のラベルがつけられていないファイルに、秘密度ラベルを付与するサンプル スクリプトです。 ファイル ポリシーでのガバナンス アクションでも同等のことができますが、障害等で、ガバナンス アクションが実行されてない場合の補完として、 手動でラベル付けを行う操作を代替するものです。 |

# Communication Compliance 関連記事
| ページ | 内容 |
| --- | ---- |
| [Communication Compalince 用語サンプル](https://github.com/YoshihiroIchinose/E5Comp/blob/main/CC.md) | コミュニケーション コンプライアンスのテスト用に定義したサンプルの要注意ワードです。 |
|[Communication Complinance のポリシーで合致した内容を一括ダウンロードする](https://github.com/YoshihiroIchinose/E5Comp/blob/main/CCExport.md)|Communication Compliance の特定のポリシーに合致したメッセージを抽出する操作を自動化するサンプル スクリプトをご紹介します。|
|[Communication Compalince 用語サンプル2](https://github.com/YoshihiroIchinose/E5Comp/blob/main/CC2.md) | コミュニケーション コンプライアンスのテスト用に定義した相場操縦・株価操作に関するワードのサンプルです。 |

# EDM 関連記事
| ページ | 内容 |
| --- | ---- |
|[機密情報や Exact Data Match の動作に関して](https://github.com/YoshihiroIchinose/E5Comp/blob/main/EDMMatching.MD)| 機密情報の検出や、Exact Data Match における日本語環境での現状の動作をまとめています。 |
|[EDM のデータ アップロード用に全角英数字・記号を半角に変換する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/EDM_Preprocess.md)|EDM のデータ アップロード用に CSV ファイルの中の全角の英数字・記号を半角に変換する PowerShell スクリプトのサンプルです。|
| [EDM での日本語住所検知用にキーワード辞書を作成する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AddressDictionayforEDM.md) | PowerShell を用いて日本郵便の提供する住所データをキーワード辞書用に整備する方法についてです。 |
| [キーワード辞書でワード マッチにおけるワードブレークのずれに対処する](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AddressDictionayforEDM2.md) | キーワード辞書での検知を行う際、本文側で丁番号が入る入らないでワードブレーク位置が異なってしまう問題の分析と、解決方法についてです。 | 
| [キーワード辞書に合わせて EDM で取り込む住所をトリミングする](https://github.com/YoshihiroIchinose/E5Comp/blob/main/AddressDictionayforEDM3.md) | 事前作成済みの住所のキーワード辞書を元に、 EDM のデータの下処理として、住所のデータをキーワード辞書に含まれる都道府県市区町村町域名までにトリミングするスクリプトを紹介しています。 |
