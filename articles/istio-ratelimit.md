---
title: "istioでratelimitをかける"
emoji: "🐓"
type: "tech"
topics: ["kubernetes", "istio", "microservice"]
published: false
---

# Global rate limitを導入する

istioを使ってAPI Rate-Limitを実装しよう。

様々なマイクロサービスの本やドキュメントには、APIには必ずRate-Limitをかけよう的なことが書いてあるが実際のところ大事なのはわかるけれど、各アプリケーションに個別に実装していったらきりがないのでつい後回しになってるといった事になりがち

istio(envoy)でRate-Limitをかけるモチベーションはアプリケーションにまったく変更を入れることなく、言語、フレームワークに依存しない統一的な行うことでRatelimitの導入を楽に導入していく。

## Overview

今回適用するRate-Limitの大体の概要図

[istioオフィシャルのサンプル](https://istio.io/latest/docs/tasks/policy-enforcement/rate-limit/)ではingress gatewayに適用して全てのトラフィックに対してRateLimitをかけているが、今回は影響範囲を小さく試すために特定のWorkloadにのみ適用する。

![overview](/images/istio-ratelimit/global-ratelimit01.png)
*global rate-limit overview*

それぞれの責務は以下のようになる。

### envoyproxy/ratelimit
  - envoyのratelimit extension implements
  - (redis)実際に誰が、どれくらいアクセスしたのか保存しておくデータストア。
  - **quoter設定管理**
  - prometheus statsd exporter

### **application(workload)**
  - envoyでHTTP requestを分析し、このrequestはどの `descriptor`に分類されるのかを決定する。(図のapp sidecar proxyのRateLimit部分)
  - **quoterは管理しない**

### ingress gateway

- 今回は基本的に何もしない、Ratelimitを実運用するのであればここにRatelimitFilterを追加するのがいいと思う


## 構成

[example manifests](https://github.com/Sho2010/istio-example/blob/main/rate-limit)
```
.
├── envoy-ratelimit # envoyproxy/ratelimit deploy ratelimit namespace
└── httpbin         # ratelimitで制限するサービス ratelimit-test namespace
```

# 実際の流れ

1. envoyが定義したRatelimit拡張I/Fの実装である [envoyproxy/ratelimit](https://github.com/envoyproxy/ratelimit)をデプロイする
1. ratelimitの対象となるhttpbinをデプロイする
1. httpbinにEnvoyFilterを適用してratelimitをかける
1. ratelimitのconfigを調整する

## 🚀 Deploy envoyproxy/ratelimit

この時点では設定を変える必要はないのでそのままデプロイする。
kustomizeを使っているが、全部のmanifestにnamespaceを設定するのがめんどくさくてそれにしか使ってないので個別にinstallしても問題ない。
prometheusなどで使うmetricsが必要ないのであれば、statsd exporterはデプロイせずにratelimitのenvを`USE_STATSD=false`に変更してもよい

```sh
$ kustomize build | kubectl apply -f -
```

## 🚀 ratelimit対象のserviceをデプロイする

とりあえず定番のDeployment + Service + Gateway + VirtualService + DestinationRule の基本構成でhttpbinをデプロイする。

サンプルの以下のファイル郡を適用していく。

- deployment.yaml
- gateway.yaml
- service.yaml
- traffic-management.yaml
- peer-auth.yaml

istioを使ってる人にとっては特に変わったこともしていないので割愛

# 🚥 ratelimitを適用する

ratelimitを適用していく。最初に少し触れたがアプリケーション側、ratelimit拡張側両方に設定を行う必要がある

- アプリケーション側でどんなリクエストに制限を行うのかDescriptorを決定する。
- ratelimit拡張側でDescriptorに対してどれくらいの流量を許容するのか決定する。

### アプリケーション側の設定(Descriptorの決定)

リクエスト情報を元に**descriptor**(設定を探すキーのようなもの)を決定する設定を記述する。

例としては以下のようなものである。

- `POST` methodに対して `post-quota`
- `/limited/*`pathに対して`limited-api-quota`
- `X-PLAN: premium` http header field に対して `premium-quota`
- etc... etc...

実際のmanifestを見ていく。

`limit.yaml` はratelimit extensionに接続するための情報が記述されてる。
いくつか注意する設定があるのでみていく

```yaml:httpbin/filter/limit.yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
 labels:
   app.kubernetes.io/name: httpbin
 name: filter-ratelimit
 namespace: ratelimit-test
spec:
 workloadSelector:
   app.kubernetes.io/name: httpbin # label used to identify pods running your applications
 configPatches:
   - applyTo: HTTP_FILTER
     match:
       context: SIDECAR_INBOUND
       listener:
         filterChain:
           filter:
             name: envoy.filters.network.http_connection_manager
             subFilter:
               name: envoy.filters.http.router
     patch:
       operation: INSERT_BEFORE
       value:
         name: envoy.filters.http.ratelimit
         typed_config:
           "@type": type.googleapis.com/envoy.extensions.filters.http.ratelimit.v3.RateLimit
           domain: "ratelimit-httpbin" # must match domain in ratelimit ConfigMap
           failure_mode_deny: true     # run plugin in fail-open mode, no limiting happens if ratelimit is unavailable
           timeout: 10s
           rate_limit_service:
             grpc_service:
               envoy_grpc:
                 cluster_name: rate_limit_cluster
             transport_api_version: V3
   - applyTo: CLUSTER
     match:
       cluster:
         service: ratelimit.ratelimit.svc.cluster.local
     patch:
       operation: ADD
       value:
         name: rate_limit_cluster
         type: STRICT_DNS
         connect_timeout: 0.25s
         lb_policy: ROUND_ROBIN
         http2_protocol_options: {}

         load_assignment:
            cluster_name: rate_limit_cluster
            endpoints:
            - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: ratelimit.ratelimit.svc.cluster.local # ratelimit Service name
                      port_value: 8081 # and port exposed by the Service
```

### `spec.workloadSelector`

このEnvoyFilterが適用されるworkloadを探すための条件を書く。
基本的にDeploymentの `selector.matchLabels`と同じように書けばOK

### `typed_config.domain`

後述するがratelimit拡張側に設定した`domain`の値と一致しなければならない。

### `typed_config.failure_mode_deny`

設定不備などでRatelimitServiceに接続不可能だった場合の挙動を定義する。
障害箇所が増えてしまうためfalseにするのが無難だと思う。

`true`: Ratelimit serviceに接続不可能だった場合 **HTTP requestが失敗する**
`false`: Ratelimitを無視してそのままupstreamに繋いでResponseを返す。

自分の場合はデバッグ中は動いてるかどうか分からなかったのでTrueとして動作確認ができたあとはfalseとした

### `...socket_address.address`

`match.cluster.service` の値と合わせて**ratelimit serviceのアドレスを設定する。**

これでRatelimitServiceに接続する準備ができた

- - -

# Descriptorの決定

```yaml:httpbin/filter/actions.yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: filter-ratelimit-actions
  namespace: ratelimit-test
  labels:
    app.kubernetes.io/name: httpbin
spec:
  workloadSelector:
    app.kubernetes.io/name: httpbin
  configPatches:
    - applyTo: VIRTUAL_HOST
      match:
        context: SIDECAR_INBOUND
        routeConfiguration:
          vhost:
            name: inbound|http|80 # port must be a port your Service is listening on
      patch:
        operation: MERGE
        value:
          rate_limits:
            - actions:
              - request_headers: # 👀
                  descriptor_key: service-level
                  header_name: X-SERVICE-LEVEL
            - actions:
              - header_value_match: # 👀
                  descriptor_value: "user_agent_path"
                  headers:
                    - name: :path
                      prefix_match: "/user-agent"
```



ほぼ全てが`rate_limits[]` に詰まっている。
やってきたHTTPリクエストは上から順にActionsが実行され、条件が合致した場合にdescriptor key/valueがlistへ追加されていく。

```txt:descriptor
(<descriptor_key1>, "<descriptor_value1>")
(<descriptor_key2>, "<descriptor_value2>")
```

ここで注目したいのは、actionsに`request_headers`と`header_value_match` 似たような設定があること。この2つの挙動の違いを例で見ていく。

1. `X-SERVICE-LEVEL: premium` がトップ画面`/`をリクエストした場合、headerは合致するがPATHは合致しないためdescriptorは以下のようになる。

```txt:descriptor
("service-level", "premium")
```

2. `X-SERVICE-LEVEL: premium`が `/user-agent` にアクセスした場合。
この場合は両方に合致するため、descriptorは以下のようになる。

```txt:descriptor
("service-level", "premium")
("header_match", "ua")
```

あれ？ `header_match` descriptor keyはどこから出てきたのだろう？ 実はこれ定数で、`header_value_match`を使うと必ずdescriptor keyはheader_matchとなる

- `request_headers`はヘッダーフィールドが存在した場合に追加
  - Key: 任意の設定値
  - Value: HeaderFieldの値
- `header_value_match`はヘッダーフィールドの値に対して条件を適用 合致した場合に追加する
  - Key: 固定 `header_match`
  - Value: 任意の設定値
  - ℹ http method, path, hostもこっちじゃないと取れない

最初これがさっぱりわからなくてハマりまくった。ちゃんとドキュメントを読もうと思った...

See: [HeaderValueMatch](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-msg-config-route-v3-ratelimit-action-headervaluematch)
See: [RequestHeaders](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-msg-config-route-v3-ratelimit-action-requestheaders)

:::message
これらを踏まえて、自分はだいたい以下のような指針で運用することにした

- HTTP `:path`, `:method`, `:authority` などが使いたい場合は **header_value_match**
- その他は **request_headers**
:::

Descriptorの決定方法がわかったのでいよいよDescriptorに対してのクオータを決定する！

### ratelimit拡張の設定(quotaの決定)

Descriptorに対するクオーターはratelimite serviceの[ConfigMap](https://github.com/Sho2010/istio-example/blob/main/rate-limit/envoy-ratelimit/ratelimit-config.yaml)に格納されている
サービスが決定したdescriptorに対してどれくらいのクオータを割り当てるのか記述する。


```yaml:configmap.yaml
domain: "ratelimit-httpbin"  # httpbin/filter/limit.yamlに設定したdomainと一致させる！
descriptors:
  - key: service-level
    value: "premium"
    rate_limit:
      unit: minute
      requests_per_unit: 10
  - key: header_match
    value: "user_agent_path"
    rate_limit:
      unit: minute
      requests_per_unit: 1
```

この設定で以下のような動作になる
- `X-SERVICE-LEVEL: premium` のリクエストは10req/min
 -`/user-agent`のリクエストは 1req/min

📓 ここまでくれば大体ドキュメントを読めばだいたいなんとかなるので詳細はドキュメントを参照してください。
- [オフィシャル](https://github.com/envoyproxy/ratelimit)
- [blog](https://dzone.com/articles/http-throttling-using-lyft-global-ratelimiting)
  - X-CustomPlan, X-CustomHeaderの複雑な設定がすごく参考になる
- [blog](https://dev.to/tresmonauten/setup-an-ingress-rate-limiter-with-envoy-and-istio-1i9g)


# まとめ

:::message
- envoyでratelimitを適用する場合はratelimit 拡張サービスをデプロイする
- **アプリケーション側にHTTPリクエストに内容に沿って制限をかけたい条件でDescriptorを記述する。**
- **ratelimitサービス側に各Descriptorに対するクオータを記述する。**
:::

# 📓 See also

- [Github envoyproxy/ratelimit](https://github.com/envoyproxy/ratelimit)
- [(istio doc)Enabling Rate Limits using Envoy](https://istio.io/latest/docs/tasks/policy-enforcement/rate-limit/)
- [config.route.v3.RateLimit.Action](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-msg-config-route-v3-ratelimit-action-headervaluematch)
- [my example manifests](https://github.com/Sho2010/istio-example/blob/main/rate-limit)
