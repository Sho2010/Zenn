---
title: "conftestでk8s manifestにテストを書こう"
emoji: "🎴"
type: "tech"
topics: ["kubernetes", "conftest", "rego"]
published: true
---

# はじめに

今日のソフトウェア開発においてユニットテストの重要性に関しては今更語る必要もないと思います。IaCの隆盛によってコード化されたインフラもテストしたいという需要は当然のように発生します。

**そんな状況を助けてくれるツールが[conftest](https://www.conftest.dev/)です。**

conftestはyaml, JSON, xml, dockerfile, ini, etc...etc.. 様々なフォーマットに対応しており、それらにユニットテストを書くことが可能となります。また、k8s固有の技術ではないのでこれらのファイルフォーマットの設定ファイルに広く適用することができます。

conftestは [rego言語](https://www.openpolicyagent.org/docs/latest/policy-language/#what-is-rego)で`policy`という単位でルールを記述していき、通常のユニットテストツールのようにルールに抵触した場合にエラーを返します。
これによって**設定不備をCIの自動テストの段階で検出することができます。**

### Policyの例

例えばconftestはk8s manifestに以下のような有用なテストを実行することができます。 これらは実際に私が作成したものです。

- セキュリティ要件を満たすため、コンテナは`securityContext.runAsNonRoot = true` で起動しなければならない
- pod tamplateは明示的に `ImagePullPolicy` を設定しなければならない。
- health check(live/readiness probe)を明示的に指定しなければならない。
- 環境変数に明示的に `TZ=asia/tokyo` を指定しなければならない。
- 機密情報がmanifestに直書きされていてはならない。
- DeprecatedなAPI versionが利用してはならない。
- etc...etc...

conftestではこのような例を**Policyと呼び**ユニットテストにおけるspecのように取り扱います。

# 具体的に動かしてみる

## Installation

まずは公式の手順に従いconftestのインストールを行う。

https://www.conftest.dev/install/

## Policyを書いてmanifestをテストする

以下のようなDeployment manifestがある。

```yaml:deployment.yaml
kind: Deployment
metadata:
  name: awesome-app
spec:
  selector:
    matchLabels:
      app: awesome-app
  template:
    spec:
      containers:
        - name: main
          image: sho2010/awesome-app:latest
```

このマニフェストに対して以下のようなシナリオでポリシーを書いていきたいと思う

```
Deploymentには`imagePullPolicy`が明示的に設定されていなければならない。
```

それでは早速Policyを書いてみる。

policyは`k8s.rego`というファイル名で、`policy`ディレクトリの中に配置することにする。
中身は気にせずとりあえずコピペ

```rego:policy/k8s.rego
package k8s

violation_imagePullPolicy_not_set[msg] {
	input.kind == "Deployment"
	container := input.spec.template.spec.containers[_]
	not container.imagePullPolicy
	msg := sprintf("imagePullPolicy is not found to container `%v` of Deployment `%v`", [container.name, input.metadata.name])
}
```

### policyが書けたので実行してみる

Policyの詳細は後で見ていくことにしてとりあえず実行してみる。

```sh
# --policy オプションにポリシーのディレクトリ
# --namespace はregoのpackageを指定する
$ conftest test --policy ./policy --namespace k8s --output table ./deployment.yaml

+---------+-------------------+-----------+--------------------------------+
| RESULT  |       FILE        | NAMESPACE |            MESSAGE             |
+---------+-------------------+-----------+--------------------------------+
| failure | ./deployment.yaml | k8s       | imagePullPolicy is not         |
|         |                   |           | found to container `main` of   |
|         |                   |           | Deployment `awesome-app`       |
+---------+-------------------+-----------+--------------------------------+
```

いい感じにテストが失敗した。失敗の原因として最後のmsgの部分が表示されているのがわかる。それではmanifestを修正してテストを通してみよう。

```diff:deployment.yaml
        containers:
          - name: main
            image: sho2010/awesome-app:latest
+           imagePullPolicy: Always
```

再実行

```sh
$ conftest test --policy ./policy --namespace k8s --output table ./deployment.yaml

+---------+-------------------+-----------+---------+
| RESULT  |       FILE        | NAMESPACE | MESSAGE |
+---------+-------------------+-----------+---------+
| success | ./deployment.yaml | k8s       | SUCCESS |
+---------+-------------------+-----------+---------+
```

今度は成功した！🎉

# policyの中身を読んで見る

先程実行したポリシーを最初から読んでいく。

```rego:policy/k8s.rego
package k8s

violation_imagePullPolicy_not_set[msg] {
	input.kind == "Deployment"
	container := input.spec.template.spec.containers[_]
	not container.imagePullPolicy
	msg := sprintf("imagePullPolicy is not found to container `%v` of Deployment `%v`", [container.name, input.metadata.name])
}
```

### package

```rego
package k8s
```

最初の行は任意のパッケージを指定します。パッケージ名はconftest実行時に`--namespace` オプションで指定することが可能で、入力に対して指定したpackageのポリシーのみを適用します。
また、`.` が利用可能なため、以下のようにnamespace, domainのようにグルーピングすることが可能です。

- `k8s`
- `k8s.development`
- `k8s.production`

このようにポリシーのパッケージを設定しておけば「production環境のマニフェストのみ適用したいポリシー」のような分類が可能です。

### Rule Name

```rego
violation_imagePullPolicy_not_set[msg] {
# ...
}
```

conftestはルール名のプレフィックスが`violation`, `warn` のルールをテストの対象とします。

実行時、violationルールは違反した場合は戻り値に1を返し、warnルールは違反メッセージを表示しますが戻り値は0となります。

また、この命名規則以外で関数定義を行うとヘルパー関数として利用できます。

### Rule

この部分が若干regoの癖が強い部分で、

:::message
記述したルールが全て `true`の場合にPolicy違反を報告します。
:::

慣れてないうちはこの挙動に戸惑うかもしれません。


```rego:policy/k8s.rego
input.kind == "Deployment"
container := input.spec.template.spec.containers[_]
not container.imagePullPolicy
msg := sprintf("imagePullPolicy is not found to container `%v` of Deployment `%v`", [container.name, input.metadata.name])
```

このルールの概要は

1. "kind: Deployment" である
2. "spec.template.spec.containers" どれかで
3. imagePullPolicyが設定されていない

繰り返しますが、これらが全ての条件を満たした場合に違反となります。 普段 exptected, actual, assert とか書いてると脳が戸惑うのでrego脳を自分の中に作りましょう。

最後のmsgはルールに抵触した場合に表示されるメッセージをユーザーフレンドリーな形でフォーマットしています。

### 1. kind == "Deployment" であること

"ConfigMap", "Service" などはここで`false`となり条件を満たさなくなりルール違反ではなくなる 通常のプログラミングをしてる場合、 "Deployment"でなければ正常系としてreturnを書きたくなるがじっとこらえる

### 2. 代入

```
container := input.spec.template.spec.containers[_]
```

`spec.template.spec.containers` を特殊なIterator構文で取得し、container変数に代入を行っています。

regoがユニークなところがこの `[_]` で

**関数の引数シグネチャがArrayの要素型と一致する場合、そのまま呼び出す事が可能** です。

例えば、[startswith関数](https://www.openpolicyagent.org/docs/latest/policy-reference/#strings)は (string, search) の関数シグネチャを持っていますが以下のような呼び出しが可能です。

```rego

# 全てのコンテナイメージに同じプレフィックスが設定されていることをルールとして決めたいとする。

# Iteratorをインスタンス化
container := input.spec.template.spec.containers[_]

# Iteratorに対しても(string, string) シグネチャの関数が呼び出し可能
startswith(container.image , "my-container-repository/")

# もちろん個別にも書ける
# この処理は要素数が3の場合は上の処理と同義である
startswith(input.spec.template.spec.containers[0].image , "my-container-repository/")
startswith(input.spec.template.spec.containers[1].image , "my-container-repository/")
startswith(input.spec.template.spec.containers[2].image , "my-container-repository/")

```

### 3. imagePullPolicyが設定されていないことを判定する

すべての要素に対して`not`でnull判定を行う。

### 4. msgを定義する

テストが落ちた場合のメッセージを設定する。これによってユーザーは何が間違っていたのかを判断することができる。

```rego
msg := sprintf("imagePullPolicy is not found to container `%v` of Deployment `%v`", [container.name, input.metadata.name])
```

ここまでくればあとは[regoのリファレンス](https://www.openpolicyagent.org/docs/latest/policy-reference/#built-in-functions)をみればテスト書き放題です!
導入のコツとしてはやはりちょっと癖が強い言語なのでmanifestに「特定のキーが存在する/しない」、「特定の値が設定されている」などの簡単なものから書き始めるといいと思います。


# 最後に


私はSREの立場として、チーム、組織の規模によると思いますが、k8sを導入しているのであれば最終的に組織をスケールさせるためにはクラスタ管理者だけではなくアプリケーション開発者にもある程度k8s manifestを書いてもらったほうが良いと考えています。

そして開発者にはいつでも好きに自由にmanifestを書いて、自分のアプリケーションをデプロイし、k8sのエコシステムのメリットを享受し、より素晴らしいアプリケーションを作る時間に当てて欲しいと願っています。

ただし、設定の一貫性やセキュリティは担保しなければなりません。 そのための補助輪としてconftestは強力なツールとなり得ます。

conftestで設定を自動テストすることによって以下のような多大なメリットがもたらされます。

:::message
- レビューの手間が省ける。
- うっかり設定漏れがなくなる。
- 初学者にベストプラクティスに強制的に従わせ、品質を担保し、広く知識を伝搬させることが可能となる。
- regoが書けるようになればこの記事では触れなかったが、[Open Policy Agent(OPA)](https://www.openpolicyagent.org/)など応用範囲が広がる
:::

conftest導入してみてはいかがでしょうか？

**鍛えていこうぜrego脳**

この記事の続きとしてrego書くのやっぱり難しいのでregoでのTDD、ポリシー自身にテストを書く方法とちょっとしたtipsを書きたいと思っている。

# See also

- [[_]の挙動](https://www.openpolicyagent.org/docs/latest/policy-language/#variable-keys)
- [公式のサンプル](https://www.conftest.dev/examples/)
- [rego built-in functions](https://www.openpolicyagent.org/docs/latest/policy-reference/#built-in-functions)
- [Zenn rego articles](https://zenn.dev/search?q=rego)

