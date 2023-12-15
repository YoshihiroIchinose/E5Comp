# Defender for Cloud Apps の API を利用して特定フォルダ内のファイルに秘密度ラベルを付与する
本スクリプトでは、Defender for Cloud Apps の API を利用して、特定フォルダ内のラベルがつけられていないファイルに、秘密度ラベルを付与するサンプル スクリプトです。
ファイル ポリシーでのガバナンス アクションでも同等のことができますが、障害等で、ガバナンス アクションが実行されてない場合の補完として、
手動でラベル付けを行う操作を代替するものです。<br/>
<br/>
前提として、Defender for Cloud Apps 側で、対象のフォルダ・ファイルが見えていること、ファイルが秘密度ラベル付けに対応しているファイル種別であり、ラベルがまだ付与されていないという状態である必要があります。<br/>
<br/>
原理上同じ仕組みで、アクション部分を置き換えれば、ラベルの削除や共有の解除などのアクションも実行できます。

### 環境ごとに必要な設定 (環境固有の以下の値で、スクリプト内の変数を指定のこと。)
#### 1. 64 文字の Defender for Cloud Apps のトークン
[API トークン](https://security.microsoft.com/cloudapps/settings?tabid=apiTokens) のページから取得したトークンの値<br/>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDA_Autolabel4.png" width="50%">  
#### 2. Defender for Cloud Apps API の URL
トークン取得の際に表示される URL (ラベル付けの API の利用のためには、.us3 などの地域のドメインも含める必要あり)<br/>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDA_Autolabel3.png" width="50%">  
#### 3. 対象とするアプリのインスタンス番号
ファイル ページなどからアプリをクリックした際に表示されるアプリのページの URL で /service-app/ の後に続く 5-7 桁ほどの数字<br/>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDA_Autolabel1.png" width="50%">  
#### 4. 対象とするフォルダのファイル ID
ファイル ページで、対象としたいファルダの詳細情報の中にある "ファイル ID" の値<br/>
対象としたいファイルの詳細情報から "階層を表示" を選択して、フォルダの詳細情報を表示することでも、<br/>
フォルダの　"ファイル ID" の値を確認可能<br/>
SPO/OD4B の場合、"736e3abc-13d1-44fe-9aed-3b56f9878ead|d286c00e-de8e-4eb1-9881-61bd97a69abc" のような表記<br/>
Box の場合、"239616417475" のような表記<br/>
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDA_Autolabel2.png" width="50%">
#### 5. 付与する秘密度ラベルの表示名

### 必要に応じて調整する項目
1. 一度のクエリで取得するアイテム数 (本スクリプトでは、更新日時の降順で 100 )
2. ラベル付けの対象とするファイルの更新日時の範囲 (本スクリプトでは、直近 1-24 時間前の間に更新されたファイルを対象)
   
## 本スクリプトの処理内容
1. 秘密度ラベル一覧を取得し、秘密度ラベルの ID を取得する
2. 指定されたフォルダ内のサブ フォルダを再帰的にクエリし、対象となるサブ フォルダを特定する
3. 対象となるフォルダの直下にある更新日時が対象の範囲で、秘密度ラベルが付与されていないファイルをすべて取得する
4. 対象となるファイルに対して、ラベル付け操作をキックする

### スクリプト本体
~~~PowerShell
#<--Parameters
#should be replaced by the tenant domain and URL, which can be found when you get a MDA API Token
$baseUrl="xxxxxx.us3.portal.cloudappsecurity.com"

#64 chacters, should be obtained via MDA API Token page
$Token="xxxxxxxxxxxxx"

#Label name to apply
$labelname="Confidential"

#Targeted app instance, which can be identified in a URL string as 5 digits number after "/service-app/" when you click an app name at MDA File page
$instance="20892"

#Targeted folders as a array, which can be identified at MDA File page as a "File ID" of the folder
#Folder in SPO/OD4B is like "736e3abc-13d1-44fe-9aed-3b56f9878ead|d286c00e-de8e-4eb1-9881-61bd97a69abc"
#Folder in Box is like "239616417475"
$folders=@("736e3762-13d1-44fe-9aed-3b56f9878ead|d286c00e-de8e-4eb1-9881-61bd97a608e3")
#Parameters-->

#scope of target files specified by time range of modified data and the number of files
$s=[datetimeoffset]::Now.AddHours(-24).ToUnixTimeMilliseconds()
$e=[datetimeoffset]::Now.AddHours(-1).ToUnixTimeMilliseconds()
$ResultSetSize=100

#Global variables
$targetFiles=@() 
$targetFolders=@()
$headers=@{"Authorization" = "Token "+$Token}

Function GetLabel($labelName){#For getting the label ID by a label name
	$Uri="https://"+$global:baseUrl+"/api/v1/get_rms_encryption_labels/"
	$res=Invoke-RestMethod -Uri $Uri -Method "Get" -Headers $global:headers
	Start-Sleep -Seconds 1
	foreach($l in $res.data){
		If($l.name.equals($labelName)){
			return $l.id
		}
	}
	return $null
}

Function GetFoldersRecursive($parent){#Get folders recursively under a spcified folder
	"Get folders from " + $parent
	$filter='{"parentFolder":{"eq":["'+$parent+'"]},"fileType":{"eq":[6]},"instance":{"eq":['+$global:instance+']}}'
	$batchSize=100 
	$Uri="https://"+$global:baseUrl+"/api/v1/files/"
	
	$loopcount = [int][Math]::Ceiling($global:ResultSetSize / $batchSize)
	$output=@()
	For($i=0;$i -lt $loopcount; $i++){
	    $limit=$batchSize
	    if($loopcount -1 -eq $i){$limit=$ResultSetSize % $batchSize}
	    if($limit -eq 0){$limit=$batchSize}
	    $Body=@{
    		    "skip"=0 + $i*$batchSize
		        "limit"=$limit
		        "filters"=$filter
		        "sortField"="modifiedDate"
		        "sortDirection"="desc"
		    }
	    $res=Invoke-RestMethod -Uri $Uri -Method "Post" -Headers $global:headers -Body $Body
   	    "Loop: $i, From " +$i*$batchSize +", " + $res.data.Count +" folders"
	    Start-Sleep -Seconds 1
	    $output+=$res.data
	    if($res.data.Count -lt $batchsize){break}
    }
    "Retrieved " +$output.count+" folders"
    foreach($item in $output){
        if($item.isFolder){
            $global:targetFolders+=$item._id
            GetFoldersRecursive($item._id)
        }
     }
}

Function GetFolderItems($parent){#Get recent files directly under a spcified folder
	"Get files from " + $parent
	$filter='{"modifiedDate":{"range":[{"start":'+$global:s+',"end":'+$global:e+'}]},"fileType":{"neq":[6]},"fileLabels":{"isnotset":true},'
   	$filter+='"parentFolder":{"eq":["'+$parent+'"]},"instance":{"eq":['+$global:instance+']}}'
	$batchSize=100 
	$Uri="https://"+$global:baseUrl+"/api/v1/files/"
	
	$loopcount = [int][Math]::Ceiling($global:ResultSetSize / $batchSize)
	$output=@()
	For($i=0;$i -lt $loopcount; $i++){
	    $limit=$batchSize
	    if($loopcount -1 -eq $i){$limit=$ResultSetSize % $batchSize}
	    if($limit -eq 0){$limit=$batchSize}
	
	    $Body=@{
		    "skip"=0 + $i*$batchSize
		    "limit"=$limit
		    "filters"=$filter
		    "sortField"="modifiedDate"
		    "sortDirection"="desc"
		    }
	    $res=Invoke-RestMethod -Uri $Uri -Method "Post" -Headers $global:headers -Body $Body
	    "Loop: $i, From " +$i*$batchSize +", " + $res.data.Count +" items"
　　　　　　　Start-Sleep -Seconds 1
	    $output+=$res.data
	    if($res.data.Count -lt $batchsize){break}
    }
    "Retrieved " +$output.count+" files"
    foreach($item in $output){
		Foreach($act in $item.actions){
		    If($act.task_name -eq "RmsProtectTask"){#Find files which have a RmsProtectTask action
		    	"Found non-labeled file " + $item.Name
			    $global:targetFiles+=$item 
			    break
    	   		 }
		}
	}
}

#Get the label id by a specified label name
$label=GetLabel($labelname)
if($label -eq $null){
    "Error. The spcified Label is not found."
    exit
}

#Get all subfolders under specified folders
foreach ($f in $folders){
    GetFoldersRecursive($f)
}
$allFolders=$folders+$targetFolders
"---------------"
"Total folders: "+$allFolders.count
"               "
#Get all files without a label which can be labeled in targetfolders
foreach ($f in $allFolders){
    GetFolderItems($f)
}
"---------------"
"Total files to be labeled: "+$targetFiles.count
"               "

#Kick maunal labelings for all targeted files
foreach($f in $targetFiles){
    $Uri="https://"+$global:baseUrl+"/api/v1/files/bulk_governance/"
    #Prepare request body as a text because its order matters
    $Body='{"task_name":"RmsProtectTask",'
    $Body+='"entities":[{"id":"'+$f._id+'","appId":'+$f.appId+'}],'
    $Body+='"params":{"labelId":"'+$label+'"}}'
    "Apply the label to: " + $f.Name
    $Body
    Invoke-RestMethod -Uri $Uri -Method "Post" -Headers $headers -Body $Body
    Start-Sleep -Seconds 1
}
~~~
## Azure Automation 上での実行結果のアウトプット
### Box 上のファイル
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDA_AutoLabel.png" width="50%"> 

### SPO 上のファイル
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDA_Autolabel5.png" width="50%"> 

### ラベル付けの履歴
ラベル付け操作の結果および履歴はガバナンス ログで確認可能
### SPO 上のファイル
<img src="https://github.com/YoshihiroIchinose/E5Comp/blob/main/img/MDA_Autolabel6.png" width="50%"> 

