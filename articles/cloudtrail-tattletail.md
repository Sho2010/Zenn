---
title: "CloudTrailのイベントを気軽にSlack通知する"
emoji: "👮"
type: "tech"
topics: ["aws", "security", "cloudtrail"]
published: true
---

# 概要

AWSセキュリティ監査を気軽に始めていくために手始めに
[cloudtrail-tattletail](https://github.com/psanford/cloudtrail-tattletail)というLambda用のアプリケーションを用いてAWS CloudTrailの特定イベントをSlack/emailに通知する。

- [AWS CloudTrail](https://aws.amazon.com/jp/cloudtrail/)
- [cloudtrail-tattletail](https://github.com/psanford/cloudtrail-tattletail)

CloudTrailはAWS上のあらゆるセキュリティ上のイベントが記録されるため、tattletailを利用して必要なイベントのみフィルターして通知してやる。

手始めに以下の2つのイベントをSlackに通知してみる。

- [CreateAccessKey](https://docs.aws.amazon.com/ja_jp/ja_jp/service-authorization/latest/reference/list_identityandaccessmanagement.html)
  - **IAMユーザーにアクセスキーが作成されたときに通知される**
- [StartSession](https://docs.aws.amazon.com/systems-manager/latest/APIReference/API_StartSession.html)
  - **ssmを利用して、EC2への接続時に通知される**

### 実行例

SSMでEC2にログインしたときのslack通知例
これとssm側でsession開始時に実行ログを全て保存しておくようにしておけば、誰がいつ何をしたのかはいつでもわかるようになる

![overview](/images/cloudtrail/01.png)

- - -

## 手順

1. CloudTrail証跡の作成。
1. `cloudtrail-tattletail`の設定を書く。
1. `cloudtrail-tattletail`をAWS Lambdaにデプロイする。

### CloudTrail証跡の作成

CloudTrailに[証跡の作成](https://docs.aws.amazon.com/ja_jp/awscloudtrail/latest/userguide/cloudtrail-create-a-trail-using-the-console-first-time.html) を実行して任意のS3 bucketにログが出力されるように設定を行う。

## cloudtrail-tattletail 設定

CloudTrailのイベントログはJSONで保存され、tattletailはそのログに対して**jqクエリを記述して条件を満たすと通知される**仕組みになっている。
どんなイベントが発生してどんなJSONが出力されるかはCloudTrailのイベント履歴を参照しよう。

以下の設定例は、[CreateAccessKey](https://docs.aws.amazon.com/ja_jp/ja_jp/service-authorization/latest/reference/list_identityandaccessmanagement.html), [StartSession](https://docs.aws.amazon.com/systems-manager/latest/APIReference/API_StartSession.html)をSlack通知する設定例。

`rule`にマッチさせる条件と宛先を記述して、`destination` は文字通り宛先を記述する。

```toml:tattletail.toml
# See: https://github.com/psanford/cloudtrail-tattletail#configuration-example
[[rule]]
name = "Create AccessKey"
jq_match = 'select(.eventName == "CreateAccessKey")'
description = "A new Access Key has been created"
destinations = ["Slack Audit"]

[[rule]]
name = "SSM Start Session"
jq_match = 'select(.eventName == "StartSession") | "username: \(.responseElements.sessionId) target: \(.requestParameters.target)"'
description = "SSM start session"
destinations = ["Slack Audit"]

[[destination]]
id = "Slack Audit"
type = "slack_webhook"
# 通知希望チャンネルのWEBHOOK
webhook_url = "___YOUR_SLACK_WEBHOOK___"
```

## deploy

### Create Lambda

**S3 bucket triggerでLambdaを作る。対象はCloudTrailがログを出力しているs3 bucket**

Lambdaで利用するIAMにS3 bucketに権限付与するのを忘れないように。`s3:ListBucket`,`s3:GetObject`だけで動作する。

```
"Action": ["s3:ListBucket","s3:GetObject" ]
```

### Build and Deploy

cloudtrail-tattletailに[Makefileが用意されている](https://github.com/psanford/cloudtrail-tattletail/blob/main/Makefile)ので適当にbuildする。
ルールが記述された`tattletail.toml` はLambdaのZIPファイルに同梱するか、S3_CONFIG_BUCKET, S3_CONFIG_PATH環境変数で指定できるので好きな方を使う。

buildしたら成果物のZIPをlambdaにアップロードすればできあがり。あとは要件に合わせて必要なCloudTrailのイベントや通知先をを追加していけばよい。

- - -

### `role_session_name`

おまけ

SSM StartSessionの通知はassumed role していた場合に `.responseElements.sessionId`の値が`botocore-session-1234...` のような名前になってイマイチ使い勝手が悪い。

これをわかりやすくするためには、
`~/.aws/config` に[role_session_name](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-configure-role.html#cli-configure-role-session-name)を明示的に指定してやる。

```ini:~/.aws/config
[profile production-role]
role_session_name = sho2010
source_profile = default
```

こうすることによってAssumedRoleで同じRoleを共有していても、個別にセッション名を指定する事が可能となる。

セキュリティ要件としてssmでEC2に接続する人はこの設定を必須としてしまうのがいいだろう。

[sts:RoleSessionNameを利用してシステム側で強制することも可能](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_iam-condition-keys.html#condition-keys-sts)

See:
- [監査を容易にするためのロールセッション名の指定](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-configure-role.html#cli-configure-role-session-name)
- [AWS CLIがAssumeRoleする際のセッション名を指定する](https://dev.classmethod.jp/articles/aws-cli-assume-role-with-session-name/)



