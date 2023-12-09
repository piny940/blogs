## はじめに
この記事はお家kubernetes環境を作ろうシリーズの1つです。
前回はGrafana + Prometheusを導入してクラスタ監視基盤を整備しました。

https://qiita.com/piny940/items/831d386919ec8a81dae2

今回はfluxを導入して自動デプロイ基盤を整えていこうと思います。

完成品はこちらのレポジトリで公開しています。
https://github.com/piny940/infra

:::note warn
今回はイメージをpublicに公開している前提で設定を書いています。公開していないイメージに対して自動デプロイをする場合は適宜secretsなどの登録が必要になります。
:::

## 環境
lemonやlimeはマシン名です。VPS2台のクラスタで動かしています。
- lemon: Kagoya Cloud VPS 2コア 2GB
- lime: Kagoya Cloud VPS 2コア 2GB

- デプロイツール: kubeadm
- CRI: cri-dockerd
- CNI: Weave net

## 前提条件
- 動くDocker ImageがDocker Hubにpushされている
- kubernetesクラスタが動作している

## fluxをインストール
[ドキュメント](https://fluxcd.io/flux/installation/#install-the-flux-cli)に従ってインストールをしていきます。
```
curl -s https://fluxcd.io/install.sh | sudo bash
```

タブ補完が使えるようにします。
```
. <(flux completion bash)
```

GithubのPersonal Access Tokenを作成します。
参考: https://docs.github.com/ja/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens

トークンを環境変数に格納
```
$ export GITHUB_TOKEN=<gh-token>
```

`flux bootstrap`を実行
```
$ flux bootstrap github \
  --token-auth \
  --owner=my-github-username \
  --repository=my-repository-name \
  --branch=main \
  --path=clusters/my-cluster \
  --personal
```

これでGithubレポジトリの変更をfluxが自動検知してクラスタに適用してくれるはずです。

## イメージを自動更新できるようにする
[ドキュメント](https://fluxcd.io/flux/guides/image-update/)に従い設定を行っていきます。
こちらの記事がとても参考になりました。↓

https://qiita.com/SNakano/items/bfa253eea2176854bea1

トークンとGithubユーザー名を環境変数に設定
```
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

flux bootstrapを再実行
```
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=$GITHUB_USER \
  --repository=flux-image-updates \
  --branch=main \
  --path=clusters/my-cluster \
  --read-write-key \
  --personal
```

次に`ImageRepository`と`ImagePolicy`を設定します。
`ImageRepository`はビルドイメージを「どこから取ってくるか」を設定します。今回は練習のためDocker Hubを使う前提の設定を書きますが、ghcr.ioなどを使う場合はここの設定を変更します。

`ImagePolicy`は取ってきたイメージを「どう適用するか」を表しています。Dockerイメージについているタグ(v1.0.1とか)を見て使用するイメージをフィルタリングしたり、どのタグのイメージを最新のイメージとして扱うかを定義したりします。

これらの設定はアプリケーションごとに設定をします。

ImageRepository↓
```hello-node/image-repository.yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: hello-node
  namespace: flux-system
spec:
  image: {DockerHubアカウント名}/hello-node
  interval: 1m0s
```
DockerHubを使用する場合、imageは`{DockerHubアカウント名}/hello-node`の形で書きます。

ImagePolicy↓
```hello-node/image-policy.yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: hello-node
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: hello-node
  filterTags:
    pattern: "^[0-9]+$"
  policy:
    numerical:
      order: asc
```
タグ名は今回は`yyyyMMddHHmmss`の形で書くことにしたため、数字のタグのみにフィルタをし、昇順で最新のタグを見ています。タグの一部の文字列だけを見て最新のタグを判定したい場合はextentを使います。
例↓
https://fluxcd.io/flux/guides/image-update/#imagepolicy-examples

最後に`ImageUpdateAutomation`を設定します。これは1つのクラスタに1つ書けばいい設定になります。
参考: https://fluxcd.io/flux/guides/image-update/#configure-image-updates

```image-update.yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 30m
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@users.noreply.github.com
        name: fluxcdbot
      messageTemplate: '{{range .Updated.Images}}{{println .}}{{end}}'
    push:
      branch: main
  update:
    path: ./clusters/my-cluster
    strategy: Setters
```

または以下のコマンドで自動生成できます。
```
$ flux create image update flux-system \
--interval=30m \
--git-repo-ref=flux-system \
--git-repo-path="./clusters/my-cluster" \
--checkout-branch=main \
--push-branch=main \
--author-name=fluxcdbot \
--author-email=fluxcdbot@users.noreply.github.com \
--commit-template="{{range .Updated.Images}}{{println .}}{{end}}" \
--export > ./clusters/my-cluster/flux-system-automation.yaml
```

これでイメージの変更を検知する準備は出来たのですが、最後に「変更を検知したらどの行を書き換えるのか」を指定する必要があります。
アプリの`deployment.yaml`の`spec.template.spec.containers[0].image`に次のようにマーキングをつけます。

```
image: {DockerHubアカウント名}/{レポジトリ名}:{タグ} # {"$imagepolicy": "flux-system:hello-node"}
```

これでイメージをpushしたら自動でcommitが作成されるはずです。

## 最後に
今回はfluxを用いてDockerイメージをpushしたら自動でデプロイされるようにしました。
次回はDockerイメージのpushも自動でできるよう基盤を整えていきたいと思います。

## 参考資料

https://fluxcd.io/flux/guides

https://qiita.com/SNakano/items/bfa253eea2176854bea1

https://qiita.com/zembutsu/items/1effae6c39ceae3c3d0a

https://docs.github.com/ja/actions/publishing-packages/publishing-docker-images