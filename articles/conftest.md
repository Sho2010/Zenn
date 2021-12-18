---
title: "conftest"
emoji: "📌"
type: "tech"
topics: ["opa", "conftest"]
published: false
---



```reco
package k8s.kustomize

# 単純にcontainsで調べるkey郡
sensitive_key_list = [
	"DATABASE_URL",
	"GRANT",
	"PASSWORD",
	"SALT",
	"SECRET",
	"SENTRY_DSN",
	"SLACK_WEBHOOK",
	"_PASS",
	"PASSWD",
]

# 正規表現で引っ掛けるkey郡
sensitive_regexp_list = [".*_TOKEN$"]

deny_sensitive_data_included_to_kustomization[msg] {
	contains(input.configMapGenerator[_].literals[_], sensitive_key_list[_])
	msg = "kustomization.yamlに機密情報が記述されています"
}

deny_sensitive_data_regexp_match_to_kustomization[msg] {
	regex.match(sensitive_regexp_list[_], split(input.configMapGenerator[_].literals[_], "=")[0])
	msg = "kustomization.yamlに機密情報が記述されています"
}
```

