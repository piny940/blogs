## はじめに

前回は external-dns を改良して kubernetes で作成した ingress の DNS 設定が自動でされるようにしました。

https://qiita.com/piny940/items/ca1b1ccced83708e968a

今回は kubernetes で ingress を作成したら自動で cloudflare tunnel に DNS の設定が追加されるようにしていきたいと思います。

完成品は Github 上に公開してありますので参考程度にどうぞ
https://github.com/piny940/external-dns
