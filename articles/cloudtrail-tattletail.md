---
title: "CloudTrailのイベントをSlackに通知する"
emoji: "✍"
type: "tech"
topics: ["aws", "security", "cloudtrail]
published: false
---

cloudtrailの特定イベントをSlack/emailで通知する

- https://aws.amazon.com/jp/cloudtrail/
- https://github.com/psanford/cloudtrail-tattletail

### 設定

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

# slack#bot-test
[[destination]]
id = "Slack Audit"
type = "slack_webhook"
webhook_url = "___YOUR_SLACK_WEBHOOK___"
```

### `role_session_name`

SSM StartSessionの通知はassumed role していた場合に `.responseElements.sessionId`の値がbotocore-session-1234... のような名前になってイマイチ使い勝手が悪い。

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

### 実行結果

![overview](/images/cloudtrail/01.png)

