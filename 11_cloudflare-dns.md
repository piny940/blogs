## はじめに

前回は自己署名証明書を用いて TLS 通信をできるようにしました。

https://qiita.com/piny940/items/ca1b1ccced83708e968a

今回は kubernetes で ingress を作成したら自動で cloudflare tunnel に DNS の設定が追加されるようにしていきたいと思います。

完成品は Github 上に公開してありますので参考程度にどうぞ
https://github.com/piny940/external-dns

## 方針

外部の DNS を自動で設定するアプリケーションはすでに[external-dns](https://github.com/kubernetes-sigs/external-dns)で実現されていました。external-dns は cloudflare にも対応していたのですが、cloudflare tunnel には対応していなかったので、その差分を対応することにしました。

cloudflare DNS から cloudflare tunnel に変えるには、具体的には A レコードの追加/削除部分を書き換える必要があります。

A レコードが追加される際には、

- cloudflare tunnel の設定(configuration)の ingress に新しいレコードを追加([参考](https://developers.cloudflare.com/api/operations/cloudflare-tunnel-configuration-put-configuration))
- cloudflare DNS に、`{TUNNEL_ID}.cfargotunnel.com`への CNAME レコードを追加

の 2 つをする必要があります。
