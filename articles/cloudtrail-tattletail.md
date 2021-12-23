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

### 実行結果

![overview](/images/cloudtrail/01.png)

