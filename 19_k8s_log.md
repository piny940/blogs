## やりたいこと

kubernetes の各 POD の標準出力を収集し、Pod や Node が削除されてもログが保持され続ける、サーバーに SSH で入らなくてもログを UI 上で確認できるようにする。  
無料のログ集積サービス・ログ解析ツールと連携できる
ログを見るときにログレベルで表示を分けたい

## ログ収集

- Fluentd
- Logstash
- Apache Flume

|            | Fluentd                                | Logstash             | Apache Flume                 |
| ---------- | -------------------------------------- | -------------------- | ---------------------------- |
| メリット   | 日本語の参考記事が多い・採用企業が多い | Fluentd よりシンプル |
| デメリット |                                        |                      | メンテ頻度が微妙・設定が複雑 |

日本語記事の多さ・採用企業の多さから Fluentd に

## ログ可視化

- Grafana
- Kibana

メトリクス監視に Grafana を使っていた。
ログ監視にまで Grafana を使うのはレアケースだ(普通ログ監視は Kibana、メトリクスは Grafana って使い分けをすることが多い)が、メトリクス監視ツールとログ監視ツールが分かれるのはめんどくさいので、今回は Grafana を使う。

## ログ集積

ログ可視化に Grafana を使う場合、ログ集積には Loki を使うのが一般的。だが、Loki はログメッセージに対してインデックスを張らないため、検索がもっさりするという記事を見かけた(https://zenn.dev/haccht/articles/28beda62a864d93d23e2)。そのため、今回は Fluentd でログを集め、Elasticsearch に保存することにする。

## fluentd の設定

https://github.com/fluent/fluentd-kubernetes-daemonset/issues/412

## 参考資料

https://note.com/shift_tech/n/n503b32e5cd35

https://qiita.com/zuzu0301/items/d2b46c817d97f330d91d

https://qiita.com/MetricFire/items/db7cb02e71f5ad09c304

https://medium.com/@Oskarr3/feeding-loki-with-fluentd-4e9647d23ab9

https://tech.griphone.co.jp/2019/12/14/advent-calendar-20191214/

https://docs.fluentd.org/container-deployment/kubernetes

https://zenn.dev/haccht/articles/28beda62a864d93d23e2

https://github.com/fluent/fluentd-kubernetes-daemonset/issues/412
