---
title: "RegoのPolicy test fixtureにyamlを使う"
emoji: "📌"
type: "tech"
topics: ["opa", "rego", "conftest"]
published: false
---

# tl;dr

:::message
opa test実行時のinputにyamlを使いたい場合

**yaml.unmarshal(yaml_str)**
:::

```rego:example_test.rego
test_imagePullPolicySet_not_set_on_deployment {
	manifest = `
		kind: Deployment
		metadata:
			name: some-app
`
	input := yaml.unmarshal(manifest)

	warn_awesome_test["expected message"] with input as input
}
```


# Summary


```sh
$ opa test . -v
```

今日のソフトウェア開発においてユニットテストの重要性に関してはもう今更語る必要もないでしょう。
conftestの出現によって、yamlに対してユニットテストの実行が可能となりました。これによって設定不備をCIの自動テストの段階で検出することができます。

主にk8sの設定しており
見落としがちな設定などをテストによって担保することが可能です。

初学者や、チームの様々なスキル
ベストプラクティスに自然と従うことが可能となります。

例えばconftestでk8s manifestに以下のような有用なテストを実行することができます。

- pod tamplateは明示的に `ImagePullPolicy` を設定しなければならない
- セキュリティ要件を満たすため、プライマリアプリケーションは`containers[].securityContext.runAsNonRoot = true` で起動しなければならない
- 環境変数に明示的に `TZ` を指定しなければならない。
- etc...etc...


ただし、そのユニットテストの正しさを担保するためのテスト
regoは若干クセのある言語なので初学者にとって自由に
実際のyamlをユニットテストを書きながら、




policyにも



```rego
warn_imagePullPolicy_not_set[msg] {
	input.kind == "Deployment"

	container := input.spec.template.spec.containers[_]
	not container.imagePullPolicy
	msg := sprintf("imagePullPolicy is not set to container %v of Deployment %v", [container.name, input.metadata.name])
}
```

```rego
test_imagePullPolicySet_not_set_on_deployment {
	manifest = `
    kind: Deployment
    metadata:
      name: some-app
    spec:
      template:
        spec:
          containers:
            - name: nginx
              image: nginx:latest
  `

	input := yaml.unmarshal(manifest)

	warn_imagePullPolicy_not_set["imagePullPolicy is not set to container nginx of Deployment some-app"] with input as input
}
```


```rego
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
