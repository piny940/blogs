## はじめに

院試があってここ最近開発ができていなかったので、久しぶりの記事投稿になります。

今回は Kubernetes でいい感じにログを収集・可視化する仕組みを整えたので紹介していこうと思います。

## やりたいこと

現状はログを確認するために毎回サーバーに SSH でログインして `kubectl logs` でログを確認しています。  
しかしこの方法だと、

- Pod が削除されるとログを確認できなくなる
- 細かいログ検索ができない

という問題があります。

そこで、

- kubernetes の各 Pod の標準出力を収集し、Pod や Node が削除されてもログが保持され続けるようにする
- サーバーに SSH で入らなくてもログを UI 上で確認できるようにする
- 無料のログ集積サービス・ログ解析ツールと連携する

ということをやっていきます。

## 技術選定

ログ周りで使う技術は大きく分けて以下の 3 つがあります。

- ログ収集
- ログ集積
- ログ解析

ここではそれぞれで使うツールを選定していきます。

### ログ収集

ログ収集のツールとして以下の 3 つを比較してみました。

- Fluentd
- Logstash
- Apache Flume

|            | Fluentd                                | Logstash             | Apache Flume                 |
| ---------- | -------------------------------------- | -------------------- | ---------------------------- |
| メリット   | 日本語の参考記事が多い・採用企業が多い | Fluentd よりシンプル |
| デメリット |                                        |                      | メンテ頻度が微妙・設定が複雑 |

日本語記事の多さ・採用企業の多さから Fluentd を使うことにしました。

### ログ解析

ログ解析のツールとして以下の 2 つを比較しました。

- Loki
- Kibana

|            | Loki                                                  | Kibana                |
| ---------- | ----------------------------------------------------- | --------------------- |
| メリット   | メトリクス監視に使ってる Grafana にそのまま埋め込める | UI が使いやすい(主観) |
| デメリット |                                                       |                       |

これに関しては、はじめは Kibana だけを使おうと思っていましたが、Grafana に埋め込めるという点が魅力的だったので、両方を使えるようにして、用途に応じて使い分けることにしました。

この記事では Kibana でログを確認する方法を紹介していこうと思います。

### ログ集積

調べたところ、Elasticsearch や InfluxDB、Datadog などがよく使われるようでした。  
今回はログ解析に Kibana を使うことを決めていたので、連携がしやすい Elasticsearch を使うことにしました。  
(EFK スタックという言葉があるらしく、参考資料が豊富で良さそう)

## 実装

### ES・Kibana

まずは ElasticSearch と Kibana を設定していきます。

[Elastic Cloud on Kubernetes(ECK)](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html)というものがあるので、これを使います。

ECK を使うと、Kibana から Elastic Search へ接続するのに使う Secret が自動で作成されるので、便利です。

完成したコードはこちらから確認できます。

https://github.com/piny940/infra/tree/main/kubernetes/apps/elastic-system

ポイントは以下の 2 つです。

**Elastic Search で Volume サイズを明記**

ES はデフォルトで 1GB のボリュームを確保しますが、1GB だとすぐに足りなくなってしまうため、あらかじめサイズを明記しておきます。

参考：https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-volume-claim-templates.html

```yaml
volumeClaimTemplates:
  - metadata:
      name: elasticsearch-data # Do not change this name unless you set up a volume mount for the data path.
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
      storageClassName: # Your storage class
```

**Kibana で publicBaseUrl を設定**

Kibana では、publicBaseUrl を指定しておかないと警告が出るので、設定しておきます。

```yaml
config:
  server.publicBaseUrl: # Your Kibana Url
```

参考：

https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-kibana-advanced-configuration.html#k8s-kibana-configuration

https://www.elastic.co/guide/en/kibana/current/settings.html

### Fluentd

次に Fluentd を設定していきます。

Fluentd は Kubernetes 上で動かせる Daemonset が提供されているので、それを使います。

https://github.com/fluent/fluentd-kubernetes-daemonset

この中に、fluentd と Elastic Search を連携する例があるので、それを参考に設定していきます。

https://github.com/fluent/fluentd-kubernetes-daemonset/blob/master/fluentd-daemonset-elasticsearch.yaml

完成したコードはこちらから確認できます。

https://github.com/piny940/infra/tree/main/kubernetes/apps/fluentd

私は containerd を使っているため、fluentd の設定として以下の 2 つを追加しました。

**dedot_filter.conf**

label で`kubernetes.labels.app`のようなラベルが付与されるのですが、ES ではこのようなラベルだとエラーが生じていたため、de_dot の設定を追加しました。(参考：https://github.com/fluent/fluent-bit/issues/4386)

```conf
<filter kubernetes.**>
  @type             dedot
  de_dot            true
  de_dot_separator  _
  de_dot_nested     true
</filter>
```

**tail_container_parse.conf**

fluentd-kubernetes-daemonset ではデフォルトでログを JSON として処理するのですが、containerd では `YYYY-MM-DD hh:mm:ss stdout F xxxxx`の形式のログを出力するため、この形式のログを parse できるように設定します。(参考：https://github.com/fluent/fluentd-kubernetes-daemonset?tab=readme-ov-file#use-cri-parser-for-containerdcri-o-logs)

```conf
<parse>
  @type "regexp"
  expression /^(?<time>[^ ]+) (?<stream>stdout|stderr) (F|P) (?<log>.*)$/
</parse>
```

---

設定ファイルの追加方法は[こちら](https://github.com/fluent/fluentd-kubernetes-daemonset?tab=readme-ov-file#use-your-configuration)から確認できます。

設定ファイルから ConfigMap を作成し、それを Volume として Pod にマウントします。

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
(略)
configMapGenerator:
  - name: config-volume
    files:
      - conf/tail_container_parse.conf
      - conf/dedot_filter.conf
```

```yaml
# daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
(略)
spec:
  template:
    spec:
      containers:
        - name: fluentd
          (略)
          volumeMounts:
            - name: config-volume
              mountPath: /fluentd/etc/tail_container_parse.conf
              readOnly: true
              subPath: tail_container_parse.conf
            - name: config-volume
              mountPath: /fluentd/etc/conf.d/dedot_filter.conf
              readOnly: true
              subPath: dedot_filter.conf
              (略)
      volumes:
        - name: config-volume
          configMap:
            name: config-volume
        (略)
```

これで無事ログを Elastic Search に流すことができるようになります。

## 動作確認

Kibana に接続するための Ingress を作成し、URL にアクセスします。

![スクリーンショット 2024-08-18 11.32.02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/84172d05-c2ac-2bed-ba9e-c5b75d8a1b58.png)

ユーザー名は `elastic`、

パスワードは以下のコマンドで確認できます。(参考：https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-elasticsearch.html)

```sh
PASSWORD=$(kubectl get secret quickstart-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
```

`Analytics > Discover` の Date View から `Create new view` を選び、 index に`logstash-*` を指定します。

![スクリーンショット 2024-08-18 11.38.40.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/87b4cb39-9592-447f-271e-7ca6c9e3c093.png)

これで fluentd から送られてくるログを閲覧できます。

## まとめ

今回は Kubernetes の Pod で生じたログを収集・解析する方法を紹介しました。
fluentd の設定方法について学べて、いい勉強になりました。

次回は Alertmanager を使って、サービスに異常が生じた際 Slack に alert が流れる仕組みを作っていこうと思います。

## 参考資料

https://note.com/shift_tech/n/n503b32e5cd35

https://qiita.com/zuzu0301/items/d2b46c817d97f330d91d

https://qiita.com/MetricFire/items/db7cb02e71f5ad09c304

https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-volume-claim-templates.html
