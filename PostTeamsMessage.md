# Graph API を利用して Teams のチャネル チャットに投稿する
CC / DLP のテストなどに利用するため、Teams のチャネル チャットへの投稿を自動化するスクリプトのサンプルです。管理者による同意を不要としたバージョンは[こちら](https://github.com/YoshihiroIchinose/E5Comp/blob/main/PostTeamsMessage2.md)。

## 事前準備
1. Azure AD (https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps) でアプリケーションを登録する
2. 作成したアプリケーションの API のアクセス許可で、Group.Read.All、Channel.ReadBasic.All、ChannelMessage.Send の権限を与え、管理者の同意を実施する
3. 事前準備したアプリケーションの Client Secret を新規作成する
4. 以下のスクリプトで、作成したアプリケーションの ClientID、Client Secret、ユーザーの ID/パスワード、および投稿先のチーム名、チャネル名、投稿内容を書き換える
## スクリプト本体
Graph API では、ID での指定が必要になるため、チーム名からチームを、チャネル名でチャネルを特定してから投稿する処理となっています。
チームやチャネルの ID が分かっている場合、"#チーム ID の特定" および "#チャンネル ID の特定" はスキップ可能です。

````
$clientID = "xxxx"
$clientSecret="xxxxx"
$Id="xxxx@xxxx.onmicrosoft.com"
$Password = "xxxx"

$TeamName="Sales and Marketing"
$ChannelName="General" #日本語で一般と表示されている場合でも General の指定が必要
$Text="今日はいい天気ですね。"

#トークンの取得
$tokenBody = @{  
    Grant_Type = "password"
    client_Id  = $clientId
    client_secret  = $clientSecret
    username   =  $Id
    password   = $Password
    resource  = "https://graph.microsoft.com"
}

$tokenResponse = Invoke-RestMethod "https://login.microsoftonline.com/common/oauth2/token" -Method Post  -Body $tokenBody -ErrorAction STOP

$headers = @{
    "Authorization" = "Bearer $($tokenResponse.access_token)"
    "Content-type"  = "application/json"
}

#チーム ID の特定
$url="https://graph.microsoft.com/v1.0/groups?`$filter=displayName+eq+'$TeamName'"
$r=Invoke-RestMethod -Method GET -Uri $url -Headers $headers
if(!$r.value){"The group is not found";return}
$TeamID=$r.value[0].id

#チャンネル ID の特定
$url="https://graph.microsoft.com/v1.0/teams/$TeamID/channels?`$filter=displayName+eq+'$ChannelName'"
$r=Invoke-RestMethod -Method GET -Uri $url -Headers $headers
if(!$r.value){"The channel is not found";return}
$ChannelID=$r.value[0].id

#投稿先
$url="https://graph.microsoft.com/v1.0/teams/$TeamID/channels/$ChannelID/messages"

#投稿内容
$body= @"
{"body": {contentType: "text",content:"$Text"}}
"@

#投稿 日本語を投稿する際は、-ContentType "application/json; charset=utf-8" の指定が必須
Invoke-RestMethod -Method POST -Uri $url -Body $body -Headers $headers -ContentType "application/json; charset=utf-8"
````
