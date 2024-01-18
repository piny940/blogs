## はじめに
この記事は「お家Kubernetes環境を作ろう」シリーズの1つです。前回はGrafana + Prometheusを導入してクラスタ監視基盤を整備しました。

https://qiita.com/piny940/items/831d386919ec8a81dae2

今回はfluxを導入して自動デプロイ基盤を整えていこうと思います。

完成品はこちらのリポジトリで公開しています。
https://github.com/piny940/infra

:::note warn
今回はイメージをpublicに公開している前提で設定を書いています。公開していないイメージに対して自動デプロイをする場合は適宜secretsなどの登録が必要になります。
:::

## 環境
lemonおよびlimeはマシン名です。VPS2台のクラスタで動かしています。
- lemon: Kagoya Cloud VPS 2コア 2GB
- lime: Kagoya Cloud VPS 2コア 2GB

- デプロイツール: kubeadm
- CRI: cri-dockerd
- CNI: Weave Net

## 前提条件
- 動作するDockerイメージがDocker Hubにプッシュされている
- Kubernetesクラスタが動作している

## fluxのインストール
[ドキュメント](https://fluxcd.io/flux/installation/#install-the-flux-cli)に従ってインストールを行います。
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

タブ補完が使えるようにします。
```bash
. <(flux completion bash)
```

GitHubのPersonal Access Tokenを作成します。
参考: [個人用アクセス トークンを管理する](https://docs.github.com/ja/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)

トークンを環境変数に格納します。
```bash
$ export GITHUB_TOKEN=<gh-token>
```

次に、`flux bootstrap`コマンドを実行します。
```bash
$ flux bootstrap github \
  --token-auth \
  --owner=my-github-username \
  --repository=my-repository-name \
  --branch=main \
  --path=clusters/my-cluster \
  --personal
```

これでGitHubリポジトリの変更をfluxが自動検知してクラスタに適用してくれるはずです。

## イメージの自動更新設定
[ドキュメント](https://fluxcd.io/flux/guides/image-update/)に従って設定を行います。以下の記事も参考になります。

https://qiita.com/SNakano/items/bfa253eea2176854bea1

トークンとGitHubユーザー名を環境変数に設定します。
```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

次に、`flux bootstrap`を再実行します。
```bash
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=$GITHUB_USER \
  --repository=flux-image-updates \
  --branch=main \
  --path=clusters/my-cluster \
  --read-write-key \
  --personal
```

その後、`ImageRepository`と`ImagePolicy`を設定します。`ImageRepository`はビルドイメージを「どこから取ってくるか」を設定します。この例では、Docker Hubを使用する前提の設定を記述していますが、ghcr.ioなどを使用する場合はこの設定を変更します。

`ImagePolicy`は取得したイメージを「どう適用するか」を表します。Dockerイメージに付与されているタグ（例: v1.0.1など）を見て、使用するイメージをフィルタリングしたり、どのタグのイメージを最新のイメージとして扱うかを定義したりします。

これらの設定はアプリケーションごとに構成します。

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
Docker Hubを使用する場合、imageは`{DockerHubアカウント名}/hello-node`の形で書きます。

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

これでイメージの変更を検知する準備はできたのですが、最後に「変更を検知したらどの行を書き換えるのか」を指定する必要があります。
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