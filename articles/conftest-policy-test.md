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

[articles/conftest-introduction]()の続き
前の記事でk8sのmanifestにテストを書けるようになりました。
今度は書いたPolicyそのものの正しさを担保するためにregoでユニットテストを書きます。

regoのテストの書き方は以下のような素晴らしい記事があるので参照してもらうとして、

- [abc](https://zenn.dev/mizutani/books/d2f1440cfbba94/viewer/rego-test)
- [Conftestを用いたCIでのポリシーチェックの紹介](https://engineering.mercari.com/blog/entry/introduce_conftest/)

殆どのopa testのサンプルはtest対象のデータをyamlで記述してることが少ないです。
k8sでconftestを利用する場合は、実際に想定している`yaml`をテストデータとして利用するのがよい。



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
