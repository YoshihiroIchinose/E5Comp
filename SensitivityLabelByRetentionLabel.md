# 保持ラベルを利用した秘密度ラベルの適用
本ページでは、SharePoint Online および OneDrive for Business において、ライブラリ・フォルダ・ファイル単位の保持ラベル付けを行いつつ、保持ラベルを条件とした自動の秘密度ラベル付けの方法を紹介します。

## 本ページで紹介する活用のポイント
1. SharePoint Online のライブラリに対する既定の秘密度ラベルと異なり、保持ラベルであれば、ライブラリ単位だけではなく、フォルダ単位でも既定の保持ラベル付けができ、またライブラリ・フォルダに含まれる過去に作成されたファイルに対しても保持ラベル付けが可能
2. 自動の秘密度ラベル付けの機能では、ComplianceTag のファイル プロパティを用いることで、保持ラベルを条件とした自動の秘密度ラベル付けが可能
3. ライブラリ・フォルダ単位の既定の保持ラベル付けと、保持ラベルを用いた自動の秘密度ラベル付けを組み合わせることで、柔軟な秘密度ラベル付けが可能

図.保持ラベルにより秘密度ラベルが付与されたファイル

<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/RL_File01.png">
-----
## 前提条件
1. ライブラリ・フォルダ・ファイル単位の保持ラベル付けおよび、条件に応じた自動の秘密度ラベル付けはいずれも Information Protection & Governance 以上のライセンスが、ファイルの投稿者・編集者に必要
2. SharePoint Online では、[秘密度ラベルの有効化](https://learn.microsoft.com/ja-jp/purview/sensitivity-labels-sharepoint-onedrive-files)が必要
3. PDF ファイルに対しても秘密度ラベル付けを行いたい場合は、SharePoint Online の設定で [PDF の保護の有効化](https://learn.microsoft.com/ja-jp/purview/sensitivity-labels-sharepoint-onedrive-files#adding-support-for-pdf)が必要
4. 利用したい秘密度ラベルが定義されていて、ユーザーに発行されていること

## 保持ラベルについて
### 保持ラベル概要
保持ラベルは、保存場所である SharePoint Online および OneDrive for Business で管理される固有のプロパティであり、保持ラベルをファイルに適用することで、
保持ラベルに紐づけられた保持設定に応じた、一定期間の削除不可、一定期間の保持(削除時データ退避)、一定期間での削除など、ファイル毎のライフサイクル管理を実現します。ファイルの種類には依存しません。
ファイル・フォルダに対しては、1 つの保持ラベルしか設定できません。

### 保持ラベルの基本動作について
1. 保持ラベルは、SharePoint Online のサイトや、OneDriver for Business を指定し、保持ラベルを利用したい保存場所に対して発行しておく必要があります。
3. ライブラリ・フォルダへ既定の保持ラベルを付与した場合、そのライブラリ・フォルダに新規に作成されるアイテムだけではなく、設定時に存在している下位のフォルダ・ファイルに対しても、その保持ラベルが設定されます。
4. ライブラリ・フォルダで既定の保持ラベルを変更した場合、手動で保持ラベルを設定したアイテムを除いて、下位のフォルダ・ファイルに対しても既定の保持ラベルの変更が反映されます。
5. 上位のライブラリやフォルダで既定の保持ラベルを変更しても、下位のフォルダ・ファイルに手動で保持ラベルを設定していた場合、そのアイテムでの保持ラベルは置き換えられません。

## 秘密度ラベルについて
### 秘密度ラベル概要
秘密度ラベルは、管理者が定義するラベルであり、Office・PDF ファイルを中心に、秘密度に応じたファイルの分類および保護を行うためのものです。秘密度によっては、社外秘・営業秘密という扱いで、ファイルを暗号化し、テナントのユーザーであったり、特定グループのメンバーでないとファイルを開けないような保護をかけることができます。アプリケーション側での対応が必要となるため、拡張子が変わらず、専用の別アプリケーションが不要で、ネイティブに対応するファイル種別は Word・Excel・PowerPoint・PDF などに限定されます。1 つのファイルに付与可能な秘密度ラベルは、1 つだけです。

### SharePoint Online・OneDrive for Business での秘密度ラベルの扱い
秘密度ラベルを有効化した SharePoint Online・One Drive for Business では、秘密度ラベルにより暗号化保護された対応する Office・PDF ファイルがアップロードされた場合、内部的に秘密度ラベルによる暗号化保護は解除してファイルを管理します。これにより暗号化保護されているファイルであっても、全文検索、eDicovery、DLP による中身のチェックや、Office for the web などが、通常のファイルと同じように動作します。秘密度ラベルは、ファイルのプロパティとして管理され、ファイルを M365 Apps で開く場合や、ファイルをダウンロードする場合に、動的に秘密度ラベルの保護を適用して、SharePoint Online・OneDrive for Business の外にファイルが出る時には、秘密度ラベルによる保護が行われるようにしています。

### SharePoint Online・OneDrive for Business での自動の秘密度ラベル付け
1. SharePoint Onine のライブラリで既定の秘密度ラベルを付与した場合、以後そのライブラリで更新・アップロードされる対応ファイルに秘密度ラベルが付与されます。保存された過去のファイルに対しては、秘密度ラベルは付与されません。編集権限を持つユーザーが過去ファイルを Office for the Web や M365 Apps で開いた場合には、ラベルは適用されます。
2. 自動ラベル付けポリシーを SharePoint Online の各サイトや、各ユーザーの OneDriver for Business に展開して、条件に応じた秘密度ラベル付けを行う場合、1 日最大 10 万ファイルを対象に過去ファイルを含めてラベル付けが行えます。
3. 自動ラベル付けポリシーが適用された後、新規に作成・更新されたファイルは、編集完了後、数分の内にファイルの全文検索のインデックスが作成されるタイミングでラベル付けポリシーが評価されラベル付けが行われます。
4. 過去に保存されたファイルについては、バッチ処理となるタイマー ジョブで処理されるため、タイムリーにラベル付けは行われません。順次ラベルが適用されます。
5. ルール・自動によるラベル付けでは、手動で個別に付与された秘密度ラベルは置き換えませんが、既定・自動で付与されている秘密度ラベルと比べてより秘密度の高いラベルが、自動で付与対象となった場合には、秘密度ラベルが置き換わります。
6. SharePoint Online・OneDriver for Business で自動ラベル付けが行われる場合、ファイルの最終更新者がファイルの所有者となってラベル付けが行われます。
-----
## 設定例

### 保持ラベルの定義と発行
Purview ポータルのデータ ライフサイクル管理のページに行き、保持ラベルとして以下の 4 つを定義します。この例では、秘密度ラベルを付与することを前提に、保持ラベルに想定する秘密度ラベルの名称を組み入れたものとしています。保持ラベルの使い方として、どれくらいの期間で削除するかの選択肢を与えるため、1年、3年、無期限の種類を用意しています。ユーザー側でコントロールしやすいように、期間が来て削除対象となったファイルは、「1か月で削除」の保持ラベルに切り替わるようにしていて、削除をキャンセルしたい場合には、個別に保持ラベルを「機密情報・3年」などに変更することで、対応できるようにしています。なお、この設定では、明示的な保持は行っていないため、ユーザー自身による削除で、一定期間経過後にファイルは消失します。

|ラベル名|ラベル設定|期間|動作|
|---|---|---|---|
|1か月で削除|特定の期間の後にアクションを適用する|ラベル付け日時を起点に 1 か月|アイテムを自動的に削除する|
|機密情報・1年|特定の期間の後にアクションを適用する|ラベル付け日時を起点に 1 年|ラベルを "1か月で削除" に変更する|
|機密情報・3年|特定の期間の後にアクションを適用する|アイテムのラベル付け日時を起点に 3 年|ラベルを "1か月で削除" に変更する|
|機密情報・無期限|アイテムにラベルを付けるだけ|-|-|

### 保持ラベルのライブラリ・フォルダへの適用
Purview ポータルのデータ ライフサイクル管理のラベル ポリシーのページに行き、上記 4 つのラベルを静的スコープで、特定の SharePoint Online のサイトや、OneDrive for Business のサイトに発行します。いずれも、サイトの URL での指定が必要となります。複数のサイトをまとめて指定したい場合には、SharePoint サイトに識別可能なプロパティ設定をして、検索の管理プロパティにマッピングし、[Adaptive スコープ](https://learn.microsoft.com/ja-jp/purview/purview-adaptive-scopes)や、[Admistrative Unit](https://techcommunity.microsoft.com/blog/microsoft-security-blog/using-custom-sharepoint-site-properties-to-apply-microsoft-365-retention-with-ad/3133970) を事前に定義しておく必要があります。

SharePoint Site 例　https://xxx.sharepoint.com/sites/SPOLabelTests/   
OneDrive for Business 例 https://xxx-my.sharepoint.com/personal/user01_xxx_onmicrosoft_com   

保持ラベルが各サイトに展開されるまで、[1 日ほど待ちます](https://learn.microsoft.com/ja-jp/purview/create-apply-retention-labels?tabs=manual-outlook%2Cdefault-label-for-sharepoint#when-retention-labels-become-available-to-apply)。保持ラベルがサイトで利用可能になると、ライブラリ・フォルダ・ファイルに保持ラベルが適用できるようになります。SharePoint Online の各ライブラリでは、「列の追加」、「列の表示と非表示を切り替える」から「保持ラベル」と「秘密度」を追加することで、各アイテムの保持ラベルおよび秘密度ラベルがファイル一覧のビューで確認できるようになります。

ライブラリでの保持ラベルの設定は、ライブラリの設定の「このリストまたはライブラリ内のアイテムにラベルを適用する」のメニューから行う
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/RL_Library01.png">
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/RL_Library02.png">

フォルダ・ファイルの保持ラベル設定は、アイテムの詳細メニューから確認・設定が可能
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/RL_Folder01.png">
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/RL_File01.png">

### 自動の秘密度ラベル付けポリシーの設定
SharePoint Online・OneDrive for Business に保存されたファイルの保持ラベルは、ComplianceTag のプロパティとして参照が可能です。Purview ポータルの Microsoft Information Protection の自動ラベル付けポリシーに行き、以下の設定で、自動ラベル付けポリシーを設定します。

1. 名前: "保持ラベルからの秘密度ラベル・機密情報"
2. 自動適用するラベル: "Confidential/All Employees"
3. 管理単位: "完全なディレクトリ"
4. 場所　SharePoint サイト: "https://xxx.sharepoint.com/sites/SPOLabelTests/"
5. 場所　OneDrive アカウント "admin@xxx.onmicrosoft.com" (注: 保持ラベルの発行とは違い、ユーザー アカウントで指定する)
7. SharePoint サイトのルールの名前: "保持ラベル変換-SPO"
8. 条件: Document プロパティは「"ComplianceTag:機密情報・1年,機密情報・3年,機密情報・無期限"」 (注: 複数の値を,で区切って指定する場合、""で括って入力する必要がある)
9. OneDrive のファイルのルールの名前: "保持ラベル変換-ODB"
10. 条件: Document プロパティは「"ComplianceTag:機密情報・1年,機密情報・3年,機密情報・無期限"」(注: 複数の値を,で区切って指定する場合、""で括って入力する必要がある)
11. シミュレーション モードでポリシーを実行する

なお、秘密度ラベルで指定するプロパティは、検索で利用できる管理プロパティを利用しています。実際に、該当の SharePoint Online サイトや、OneDriver for Business にて、「ComplianceTag:機密情報・1年」や「ComplianceTag:機密*」などで、検索を行って、該当の保持ラベルが付与されたファイルがヒットすることを確認すると、自動ラベル付けの条件指定に問題ないかが確認できます。なお検索の際に複数の値で検索したい場合、「ComplianceTag:機密情報・1年 ComplianceTag:機密情報・3年」のようにプロパティの指定を複数スペースで区切って並べます。   

シミュレーション モードで、ヒットするファイルに問題がなさそうであれば、自動ラベル付けポリシーを有効化します。

----

### 動作例
動作例.1
- 「機密情報・1年」の保持ラベルが設定されたフォルダに新規 PowerPoint ファイルを作成すると、その PowerPoint ファイルにも、「機密情報・1年」の保持ラベルが適用されます
- 「機密情報・1年」の保持ラベルと、自動の秘密度ラベル付けのポリシーにより、ファイル作成・更新後、数分でそのファイルに「Confidential/All Employees」の秘密度ラベルが適用されます
- フォルダに付与された「機密情報・1年」の保持ラベルをクリアすると、フォルダ内の PowerPoint ファイルの保持ラベルもクリアされます
- フォルダ内の PowerPoint に付与された秘密度ラベルは「Confidential/All Employees」のままになります

動作例.2
- 既に Word ファイルが格納されているフォルダに、「機密情報・3年」の保持ラベルを適用すると、その既存 Word ファイにも、「機密情報・3年」の保持ラベルが適用されます
- 「機密情報・3年」の保持ラベルと、自動の秘密度ラベル付けのポリシーにより、そのファイルに「Confidential/All Employees」の秘密度ラベルが適用されます

動作例.3
- OneDrive に「機密情報・無期限」の保持ラベルを「Protected」フォルダに適用します
- PC のデスクトップ上の OneDrive アプリから「Protected」フォルダに新規 PowerPoint ファイルを作成します
- OneDrive for Business のクラウド側では、PowerPoint ファイルに「機密情報・無期限」の保持ラベルが適用されます
- PowerPoint ファイルに「Confidential/All Employees」の秘密度ラベルが適用されます

自動の秘密度ラベル付けポリシーで、ラベルが付与された場合、アクティビティ エクスプローラーに以下のようなログが記録されます。
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/RL_AE01.png">
