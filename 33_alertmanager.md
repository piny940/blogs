## はじめに

前回は fluentd・Elastic Search・Kibana を使って、Kubernetes の Pod で発生したログの収集・解析ができるようにしました。

https://qiita.com/piny940/items/d47653982670fc10c1c5

今回は、Alertmanager を使って、Pod がダウンした時などに Slack にアラートが飛ぶようにしていきたいと思います。

完成したコードはこちらから確認できます。

## やりたいこと

現状、サービスがダウンしても特にアラートが送られる仕組みが整っていません。  
そのため、「久々にサービスにアクセスしてみたらサイトが落ちてる！！」ということが度々起こっていました。

そこで今回は、サービスに異常が生じたら Slack に通知が流れるようにして、異常にすぐに気づけるような仕組みを整えていこうと思います。

## 実装

私の環境では、Prometheus のために [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) を使用していたので、それに乗せる形で設定をしていきます。

まず、Alertmanager の設定ファイルを記述します。

`alertname`が`Watchdog`の alert は、alertmanager が正常に動作しているかを監視するためのアラートなので、slack には流さないように設定しておきます。(参考：https://prometheus.io/docs/alerting/latest/configuration/)

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: slack-alert
  labels:
    alertmanagerConfig: slack-alert
spec:
  route:
    groupBy: ["job"]
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 12h
    receiver: "slack"
    routes:
      - receiver: "null"
        matchers:
          - name: alertname
            value: Watchdog
            matchType: "="
  receivers:
    - name: "null"
    - name: "slack"
      slackConfigs:
        - apiURL:
            name: alertmanager-secret
            key: alertmanager-webhook-url
          sendResolved: true
```

`alertmanager-secret` という名前の secret に Slack の Webhook URL を登録しておきましょう。

次に、helm の value に alertmanager の設定を追記していきます。

```yaml
# helm.value
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: kube-prometheus-stack
spec:
  chart:
    spec:
      chart: kube-prometheus-stack
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
  values:
    alertmanager:
      enabled: true
      alertmanagerSpec:
        alertmanagerConfiguration:
          name: slack-alert
```

これで Alertmanager の Pod が起動できます。

Alertmanager が起動できたら、Prometheus 側で Alertmanager に Alert を送る設定をしていきます。

Rule 自体は自分で考えるのが面倒だったので、[この辺り](https://samber.github.io/awesome-prometheus-alerts/rules.html#rule-kubernetes-1-18)のルールを拝借してきました。

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-not-ready
  labels:
    app: pod-not-ready
    prometheus: default
    role: alert-rules
spec:
  groups:
    - name: pod-not-healthy
      rules: (略)
```

作成したルールを Prometheus 側が読み込むように helm の value を変更します。

kube-prometheus-stack では標準で kubernetes の色々なアラートルールが作られているので、それも活用できるように設定をします。

```yaml
# helm.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: kube-prometheus-stack
spec:
  chart:
    spec:
      chart: kube-prometheus-stack
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
  values:
    prometheus:
      prometheusSpec:
        ruleSelector:
          matchExpressions:
            - key: app
              values:
                - kube-prometheus-stack # built in rules!
                - pod-not-ready
              operator: In
        scrapeInterval: 30s
```

これで Job に失敗した時や、Pod でエラーが発生した時などに Slack にアラートが流れるようになります。

## まとめ

今回は Alertmanager の設定をして、Slack にアラートが送信されるようにしました。  
これでサービスがダウンしていないかをいちいち心配しなくてもよくなりました！！

ただ、今回 Alertmanager を Kubernetes 上に立てたため、このままだとクラスタごとダウンした際などにアラートが流れず、異常に気づくことができません。

そのため次回はクラスタの外部にサービスを監視するジョブを定義して、クラスタごとダウンしてもそれを検知できるような仕組みを整えていこうと思います。

## 参考資料

https://github.com/prometheus-operator/prometheus-operator

https://samber.github.io/awesome-prometheus-alerts/rules.html#rule-kubernetes-1-18

https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

https://prometheus.io/docs/alerting/latest/alertmanager/
