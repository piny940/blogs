## はじめに
最近、VPS上で公開していた[ポートフォリオ](https://www.piny940.com)や[歌枠データベース](https://song-list.piny940.com)をDocker上で立てられるようにし、自宅サーバーへと完全移行しました。

(自宅サーバーへサーバーを移行した話はこの辺りの記事にまとめました。)

https://qiita.com/piny940/items/606a6b03c390c136c1f7

https://qiita.com/piny940/items/e41c2709692195f823b5

https://qiita.com/piny940/items/602316d913938db01c81

しかし、自宅サーバーは諸々の理由で頻繁に電源が落ちるため、kubernetesを使って冗長構成にしたいと思うようになりました。また、毎回sshでログインしてデプロイをするのが手間だったので自動デプロイの基盤をkubernetesで作りたいと思うようになり、意を決してkubernetesのお勉強を始めました！

前回はminikube上にNext.js + nginxのサーバーを立てました。

https://qiita.com/piny940/items/0feef2500c8111a520ea

今回は、前々回↓構築したVPS上のkubernetesクラスタ上にNext.js + nginxのサーバーをデプロイしようと思います。

https://qiita.com/piny940/items/3c1c10b80c7e173d527d

完成品はGithub上に公開してありますので参考程度にどうぞ
(`kubernetes/metallb`、`kubernetes/portfolio`が今回扱う対象です)。
https://github.com/piny940/infra/tree/main/kubernetes



