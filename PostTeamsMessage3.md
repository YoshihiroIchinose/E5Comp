# Microsoft Graph PowerShell SDK を通じて Teams チャネルに投稿する
本スクリプトは、Microsoft Graph PowerShell SDK を通じて Teams チャネルにメッセージを投稿するサンプルです。

## 事前準備
管理者権限の PowerShell で一度以下の 2 つのコマンドを実行する。
```
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module Microsoft.Graph
```

## Teams チャネルにチャネル チャットを投稿するコマンド例
```
$TeamName="Sales and Marketing"
$ChannelName="General"

# Teams チャネルに投稿する Function
Function Post ($text){
	$body= @"
	{"body": {contentType: "text",content:"$text"}}
	"@
	 New-MgTeamChannelMessage -ChannelId $Channel.Id -TeamId $Team.Id -BodyParameter $body
}

#対話的ログオン
Connect-MgGraph -Scopes "Team.ReadBasic.All","Channel.ReadBasic.All"

$user=(Get-MgContext).Account

#ユーザーが参加しているチーム全体から、該当のチームを取得
$Team=Get-MgUserJoinedTeam -UserId $user|?{$_.DisplayName -eq $TeamName}

#該当のチームから、該当のチャネルを取得
$Channel=Get-MgAllTeamChannel -TeamId $Team.Id -filter "displayName eq '$ChannelName'"

$message="現在は$(Get-date)です。"
Post $message
```
