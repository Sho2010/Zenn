---
title: "istioでWASMを使ってBasic認証を行う"
emoji: "📌"
type: "tech"
topics: ["kubernetes", "istio"]
published: true
---

# istioで使ってBasic認証を行う

オフィシャルで紹介されていた。WASMを使う方法を試してみる。

# 目的

istioを使って、アプリケーションに手を入れず、またnginxなどのProxyを入れることもなくistioとenvoyだけでBasic認証を実装する。


### 環境

- envoy: 1.11.2
- istio: 1.11.2
- basic-auth wasm: 1.11.0

### 注意

[オフィシャルのドキュメント](https://github.com/istio-ecosystem/wasm-extensions/tree/master/extensions/basic_auth#basic-auth-filter-user-guide) にもあるように、あくまでこの機能は現在experimentalであるため、プロダクションで利用しないほうがよいだろう。

明らかな問題点として今の所Secretやファイルから `user/password`を読み込む方法がなくEnvoyFilter manifestにベタ書きするしかない。これに関してもオフィシャルのREADMEやissueで言及されている。

そのため実験的に利用する以下のような人達向け

- Firewallなど別の方法で安全が担保されている状態でちょっとしたものを公開するとき、アプリケーションにわざわざBasic認証実装したくない人
- EnvoyFilterの基本を知りたい人
- istioでWASM拡張の機能を試してみたい人

- - -

# 実際の手順

基本的に、[オフィシャルのサンプル](https://istio.io/latest/docs/ops/configuration/extensibility/wasm-module-distribution/)があるのでそれに従って

[sample manifestsを書いた](https://github.com/Sho2010/istio-example/tree/main/basic-auth)

[httpbin](https://httpbin.org/)を`basic-auth`という名前でデプロイし、そこに対してbasic認証を行う。

## httpbinをデプロイする

とりあえず定番のDeployment + Service + Gateway + VirtualService + DestinationRule の基本構成でhttpbinをデプロイする。

サンプルの以下のファイル

- deployment.yaml
- gateway.yaml
- service.yaml
- traffic-management.yaml

istioを使ってる人にとっては特に変わったこともしていないので割愛

## EnvoyFilterを適用する

いよいよ本題でFilterはWASMの導入とWASMのコンフィグで二段階に分かれている

`basic-auth-filter.yaml` がWASMの導入でDownload先や定義Protoなんかが書いてある、これに関しては基本的にいじる部分はないのでそのまま適用してよい。
肝心なのは `config-filter.yaml` で、この中にWASMがどういった挙動を行うのかというコンフィグが書いてある。

```yaml
# NOTE: https://github.com/istio-ecosystem/wasm-extensions/tree/master/extensions/basic_auth
#
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
 name: basic-auth-config
 namespace: basic-auth
spec:
 configPatches:
 - applyTo: EXTENSION_CONFIG
   match:
     context: SIDECAR_INBOUND
   patch:
     operation: ADD
     value:
       name: istio.basic_auth
       typed_config:
         "@type": type.googleapis.com/udpa.type.v1.TypedStruct
         type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
         value:
           config:
             #... 中略
             # The configuration for the Wasm extension itself
             configuration:
               '@type': type.googleapis.com/google.protobuf.StringValue
                # 現時点ではSecretやConfigMapからここに値を設定する方法がないため、credentialsが剥き出しになってしまうことに注意する。
                # 一時的に運用したいなどクリティカルではないものに利用する
                #
                # 以下の例では https://$HOST/ip 以下のURLに対してbasic認証をかける
               value: |
                 {
                   "basic_auth_rules": [
                     {
                       "prefix": "/ip",
                       "request_methods":[ "GET", "POST" ],
                       "credentials":[ "ok:test", "YWRtaW4zOmFkbWluMw==" ]
                     }
                   ]
                 }
```

これによって、ごくごく小さな影響範囲でアプリケーションに手を入れることもなく、nginxなどを手前に用意することもなくistioだけでBasic認証をすることができた。よかったよかった
WASMの実装も読みきれる量だしちょっとした拡張なら自分でもかけそうな気がする良いサンプルだった。

- - -

### おまけtips: Filterを挿入する位置を考える


オフィシャルのサンプルと違う部分はサンプルのまま利用するとIngress Gatewayに適用され、全てのリクエストに対してWASM拡張が適用されてしまうため、
`spec.configPatches[].match.context=SIDECAR_INBOUND` に変更を行い適用する範囲をSidecar proxyのINBOUNDに絞る。

```diff
diff --git a/example/basic-auth/basic-auth-filter.yaml b/example/basic-auth/basic-auth-filter.yaml
index 0116d1a0..7073e4cd 100644
--- a/example/basic-auth/basic-auth-filter.yaml
+++ b/example/basic-auth/basic-auth-filter.yaml
@@ -7,7 +7,7 @@ spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
-     context: GATEWAY
+     context: SIDECAR_INBOUND
      listener:
        filterChain:
          filter:
```

![overview](/images/istio-basic-auth/basic-auth01.drawio.png)

### 参考資料

- [Configure an HTTP Filter with a Remote Wasm Module](https://istio.io/latest/docs/ops/configuration/extensibility/wasm-module-distribution/)
- [istio wasm-extensions github repository](https://github.com/istio-ecosystem/wasm-extensions/tree/master/extensions/basic_auth)
- [EnvoyFilter](https://istio.io/latest/docs/reference/config/networking/envoy-filter/)
- [istio Basic auth sample manifests](https://github.com/Sho2010/istio-example/tree/main/basic-auth)

