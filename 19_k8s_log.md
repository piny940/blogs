## やりたいこと

kubernetesの各PODの標準出力を収集し、PodやNodeが削除されてもログが保持され続ける、サーバーにSSHで入らなくてもログをUI上で確認できるようにする。  
無料のログ集積サービス・ログ解析ツールと連携できる
ログを見るときにログレベルで表示を分けたい

## ログ収集

- Fluentd
- Logstash
- Apache Flume

||Fluentd|Logstash|Apache Flume|
|--|--|--|--|
|メリット|日本語の参考記事が多い・採用企業が多い|Fluentdよりシンプル|
|デメリット|||メンテ頻度が微妙・設定が複雑|

日本語記事の多さ・採用企業の多さからFluentdに

## ログ集積

ElasticsearchやInfluxDB、Datadogなどがよく使われる
今回はPostgresn

## 参考資料

https://note.com/shift_tech/n/n503b32e5cd35

https://qiita.com/zuzu0301/items/d2b46c817d97f330d91d

https://qiita.com/MetricFire/items/db7cb02e71f5ad09c304
