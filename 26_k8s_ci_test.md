## はじめに

この記事は、Kubernetes のマニフェストファイルを GitHub Actions で自動テストする方法について説明します。

ArgoCD や FluxCD を使用していると、「マニフェストが main レポジトリにマージされると自動的にデプロイされる」という運用にすることが多いと思います。
しかしその際、マニフェストに誤りがあると、次のような問題が生じます。

- 再度 PR を出して修正する、という手間が発生する
- 本番環境 (or ステージング環境) で不具合が生じる危険性がある

そこで今回は、Github Actions を使用して、PR をマージする前にマニフェストファイルの正当性をテストできるようにしたいと思います。

## 技術選定

今回使用するツールの条件として、以下の条件があります。

- Custom Resource Definition (CRD) に対応していること

マニフェストの正当性を検査するツールとして、以下のツールを候補として考えました。

- [kuttl](https://github.com/kudobuilder/kuttl)
- [kubeval](https://github.com/instrumenta/kubeval)
- [kubeconform](https://github.com/yannh/kubeconform)

**kuttl**  
ドキュメントを軽く読んだ感じだと、CustomResource への対応という点で、kuttl は若干設定を追加する必要がありそうでした。
( https://kuttl.dev/docs/testing/tips.html#custom-resource-definitions )

**kubeval**  
[GitHub](https://github.com/instrumenta/kubeval)を見ると、

> no longer maintained

と書かれていて、kubeconform へのリンクがはられていました。

**kubeconform**  
ドキュメントを見た感じだと、CustomResource にも対応していそうでした。

以上の理由により今回は、kubeconform を使用することにしました。

## 実装

以下のように Github Actions を作成します。

作成にはこちらの記事を参考にさせていただきました。

https://qiita.com/araryo/items/f072a0cca0b098f02e44

kustomize を使用している場合、`kubeconform`の前に `kubesomize build` を実行する必要があります。

```yaml
name: Kubeconform
on: push
jobs:
  kubeconform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: yokawasa/action-setup-kube-tools@v0.9.2
        with:
          setup-tools: |
            kubeconform
            kustomize
          kubeconform: "0.6.6"
          kustomize: "5.4.1"
      - run: |
          for APP in $(ls kubernetes/apps/)
          do
            kustomize build kubernetes/apps/$APP |
            kubeconform -summary \
            -schema-location default \
            -schema-location 'https://raw.githubusercontent.com/piny940/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
            -output json
          done
```

`-schema-location` に CRD のスキーマを指定することで、CRD の検証も行うことができます。

## 生じた問題

公式では`-schema-location`に以下のように指定しています。

```
-schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json'
```

しかし、参照先の[CRDs-catalog](https://github.com/datreeio/CRDs-catalog)を見ると、Flux の[HelmRelease](https://fluxcd.io/flux/components/helm/helmreleases/)で使われている以下の CustomResource に対応していませんでした。

- `source.toolkit.fluxcd.io/v1`の`HelmRepository`
- `helm.toolkit.fluxcd.io/v2`の`HelmRelease`

どうも最新のバージョンにのみ対応していないようで、`source.toolkit.fluxcd.io/v1beta2`や`helm.toolkit.fluxcd.io/v2beta2`などに直すと、正常に検証できるようです。

ですが、`v1beta2`などを指定していると Flux 側で`deprecated`と警告が出てしまいます。

そのため、今回は[fork](https://github.com/piny940/CRDs-catalog)を作成して、それを読みに行くようにしました。

現在[PR](https://github.com/datreeio/CRDs-catalog/pull/321)のレビュー待ちです(2024 年 5 月 17 日現在)

## おまけ

今回 GithubAction で k8s マニフェストの正当性を検査できるようにしたついでに、[yamlfmt](https://github.com/google/yamlfmt)を使用して Yaml のフォーマットも検査するようにしました。

```yaml
name: Yamlfmt
on: push
jobs:
  yamlfmt:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "^1.22.1"
      - name: Install yamlfmt
        run: go install github.com/google/yamlfmt/cmd/yamlfmt@latest
      - name: Yamlfmt
        run: yamlfmt -lint kubernetes/
```

## まとめ

今回は、Github Actions を使用して、Kubernetes のマニフェストファイルの正当性を検査する方法を紹介しました。

ただ、現状 FluxCD が main ブランチに直接コミットを push する設定になってしまっているので、次回は FluxCD が PR を作成するよう設定を変更し、今回作成した検査が有効になるようにしたいと思います。

## 参考資料

https://github.com/yannh/kubeconform

https://qiita.com/araryo/items/f072a0cca0b098f02e44

https://mixi-developers.mixi.co.jp/kubeconform-2bb477371e06
