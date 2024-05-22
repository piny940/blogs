## はじめに

今回は FluxCD が GitHub に対して SSH でアクセスするようにする設定を行います。

## 手順

[公式ドキュメント](https://fluxcd.io/flux/installation/bootstrap/github/#bootstrap-without-a-github-pat)に SSH を使って接続する方法が記載されているので、それに従います。

以下のコマンドを使って bootstrap すれば SSH で接続できます。

```bash
flux bootstrap git \
  --url=ssh://git@github.com/<org>/<repository> \
  --branch=<my-branch> \
  --private-key-file=<path/to/ssh/private.key> \
  --password=<key-passphrase> \
  --path=clusters/my-cluster
```

以下、ハマったポイントです。

- SSH の URL はレポジトリの Clone に使う URL を指定しては**いけない**。ドキュメント通りのフォーマットで指定する
- private key へのパスは**絶対パス**で指定する。私は`/home/$(whoami)/.ssh/ed25519`と指定することでコードをコピペで使い回せるようにしました。

## なんで SSH なの？

FluxCD のドキュメントの一番上に書かれているコマンド(↓)に従うと、Flux は GitHub に対してトークンを使って HTTP でアクセスします。

```bash
flux bootstrap github \
  --token-auth \
  --owner=my-github-username \
  --repository=my-repository-name \
  --branch=main \
  --path=clusters/my-cluster \
  --personal
```

https://fluxcd.io/flux/installation/bootstrap/github/

実際、生成される`GitRepository`リソースの定義を見てみると、HTTPS の URL が指定されていることが確認できます。

```yaml
# This manifest was generated by flux. DO NOT EDIT.
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: flux-system
  url: https://github.com/piny940/infra.git
```

ですが、これには以下の問題があります。

- トークンを使うと、トークンの有効期限が切れたときに FluxCD が使えなくなる

もちろん`Tokens (classic)`を使えばトークンの有効期限を「無制限」にすることはできますが、セキュリティ的にはあまりよくありません。

そこで、今回は SSH を使って FluxCD が GitHub にアクセスするように設定しました。

## 最後に

今回は FluxCD が GitHub に対して SSH でアクセスするように設定しました。
次回は Kubernetes の Staging 環境を整えていきたいと思います。

## 参考資料

https://github.com/fluxcd/flux2/issues/2029

https://fluxcd.io/flux/installation/bootstrap/github/#bootstrap-without-a-github-pat