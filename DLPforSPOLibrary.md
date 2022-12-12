# SharePoint Online のライブラリ単位で DLP を適用・除外する方法
## 概要
SharePoint Online に対する DLP の適用では、通常サイト単位で DLP の適用先を指定するため、サイト内のライブラリでは、ライブラリによらず同様の DLP ポリシーが適用されます。ただ、実際の運用の中では、False Positve が多いライブラリだけ DLP の適用を除外したいケースや、特定のライブラリだけで DLP を動作させたいというニーズもあります。ここでは、ライブラリに対する既定の保持ラベルを DLP の条件とすることで、SharePoint Online のサイト内で、ライブラリ単位に DLP の適用や、DLP の適用除外を制御する方法について紹介します。

## 前提
### ライセンス
SharePoint Online に対する DLP のみであれば、Office 365 E3 や SharePoint Online Plan2 でカバーされる機能ですが、ここでは、ライブラリに対する既定の保持ラベルを利用するため、このサイトを利用するユーザーには、Information Protection & Governance 以上のライセンスも必要となります(Informatino Protection & Governace の前提ライセンスとして、AIP P1 のライセンスも必要。)    
[参考: 既定の保持ラベルのライセンスについて。](https://learn.microsoft.com/ja-jp/office365/servicedescriptions/microsoft-365-service-descriptions/microsoft-365-tenantlevel-services-licensing-guidance/microsoft-365-security-compliance-licensing-guidance#licensing-for-retention-label-policies)

### 保持ラベルの動作について 
1. サイト管理者がライブラリ単位に、サイトに発行された保持ラベルのいずれかを既定の保持ラベルとして設定できる。
2. ライブラリ内にファイルがアップロードされると、そのファイルにライブラリの既定の保持ラベルが適用される。
3. 一般の投稿権限のみのユーザーは、ライブラリの既定の保持ラベルは変更できないが、ファイル・フォルダに設定された保持ラベルは変更が可能。
4. ライブラリ内にフォルダを作成した際、フォルダそのものには、既定の保持ラベルは適用されないが、フォルダに個別の保持ポリシーが設定されていない場合、フォルダ内のコンテンツにも、既定のライブラリの保持ラベルが適用される。
5. ライブラリ内のフォルダに、保持ラベルを設定した場合、フォルダ配下のファイルの保持ラベルは、フォルダに設定された保持ラベルで置き換わる。

### DLP の回避について
DLP の適用有無を保持ラベルで制御するため、以下のような方法で DLP の回避ができてしまい、この方法は DLP による完全なデータ制御を行うものではなく、うっかりミスを防ぐような手立てとして位置づける必要があります。
1. サイト管理者によるライブラリの設定変更で、サイト内のライブラリの既定の保持ラベルを変更し、DLP の適用を回避できてしまう。
3. 一旦は既定の保持ラベルの設定に従った DLP の判定は行われるものの、一般のユーザーが、ファイルやフォルダの保持ラベルを変更することで、DLP でブロックされたファイルを、DLP 対象外にすることでブロックを解除できてしまう。

## 設定手順
1. Micrsoft Purview ポータルの[データ ライフサイクル管理](https://compliance.microsoft.com/informationgovernance)にアクセスする。    
    1.A. 特定のライブラリを DLP から除外するケースでは、"共有可能"といった名称で、保持の設定なく、保持ラベルを作成する。    
    1.B. 特定のライブラリだけを DLP 対象とする場合は、"共有不可"といった名称で、保持の設定なく、保持ラベルを作成する。<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/DLPbyRetention01.png"/>    
     
2. データ ライフサイクル管理の[ラベル ポリシー](https://compliance.microsoft.com/informationgovernance?viewid=labelpolicies)にて、1 で作成した保持ラベルを、特定の SharePoint Online サイトに発行する。
    
3. 2 の操作後、数時間待ち、該当の SharePoint Online サイトのライブラリにアクセスし、右上歯車のメニューから、ライブラリの設定->その他ライブラリの設定に進み、「このリストまたはライブラリ内のアイテムにラベルを適用する」を選択する。    
    3.A. このライブラリを DLP から除外するケースでは、"共有可能"の保持ラベルを既定の保持ラベルとして設定する。     
    3.B. このライブラリのみを DLP 対象とする場合は、"共有不可"の保持ラベルを既定の保持ラベルとして設定する。<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/DLPbyRetention02.png"/>     
    
4. Micrsoft Purview ポータルの[データ損失防止](https://compliance.microsoft.com/datalossprevention?viewid=policies)で、上記の特定の SharePoint Oline サイトを対象とした、DLP ポリシーを作成する。    
    4.A. DLP のルールで、特定のライブラリを DLP から除外するケースでは、"グループ"を追加し、"条件の追加"から"保持ラベル"を選択し、"共有可能"のラベルを追加する。ルールのグループの "Not" を有効化し、保持ラベルが"共有可能"となっているアイテムを除外する設定とする。その他 DLP の条件も適宜設定する。<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/DLPbyRetention03.png"/>     
    4.B. 特定のライブラリだけを DLP 対象とする場合は、"条件の追加"から"保持ラベル"を選択し、"共有不可"のラベルを追加し、保持ラベルが"共有不可"となっているアイテムのみを対象に設定する。その他 DLP の条件も適宜設定する。<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/DLPbyRetention04.png"/> 

## 保持ラベルに応じた検索
ライブラリ内で、特定の保持ラベルがついたアイテムのみを検索したい場合には、"ComplianceTag" という管理プロパティを用いて、以下の KQL で検索が可能。    
`ComplianceTag:"共有不可"`    
    
ライブラリ内で、特定の保持ラベルがついていないアイテムのみを検索したい場合には、"ComplianceTag" という管理プロパティを用いて、以下の KQL で検索が可能。    
`-ComplianceTag:"共有可能"`    
