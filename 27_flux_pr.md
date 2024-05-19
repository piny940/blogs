## はじめに

前回は Kubernetes のマニフェストのバリデーションを CI で行えるようにしました。

https://qiita.com/piny940/items/309ad967ee76aeefa4b5

しかし、現状の FluxCD の設定では、アプリケーションの Image が更新されると、その Image を使うように設定を変えるコミットが main ブランチに直接 push されてしまいます。

https://github.com/piny940/infra/commit/765884e1724d54ed5919280f444159d85d7205de

これだと FluxCD が作った差分が CI に失敗しても本番環境に適用されてしまいます。  
そこで今回は、FluxCD の設定を変更して、アプリケーションの Image が更新されたときに、その変更を適用するような Pull Request を作成するようにします。

## 手順

### FluxCD の設定変更

まず、FluxCD の設定を変更します。
基本的に Flux の[公式ドキュメント](https://fluxcd.io/flux/use-cases/gh-actions-auto-pr/)に従って進めていきます。

`ImageUpdateAutomation`リソースには、push 先のブランチを指定するフィールドがあるため、ここを書き換えます。

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: default
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@users.noreply.github.com
        name: fluxcdbot
      messageTemplate: "{{range .Updated.Images}}{{println .}}{{end}}"
    push:
      branch: fluxcd/image-update # New!
  update:
    path: ./kubernetes/apps
    strategy: Setters
```

これで FluxCD は変更を Staging ブランチに push するようになります。

### Github Actions の設定

次に、staging ブランチに変更が push されたら自動で PullRequest が作成・マージされるように、CI を作成します。

`.github/workflows/auto-pr.yaml`を作成し、次のように記述します。

```yaml
name: FluxCD Auto-PR
on:
  push:
    branches:
      - fluxcd/image-update
jobs:
  create-and-merge-pr:
    name: Open PR to main
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        name: checkout
      - uses: repo-sync/pull-request@v2
        name: Create pull request
        with:
          destination_branch: "main"
          pr_title: "FluxCD Image Update Automation PR"
          pr_body: ":crown: *An automated PR*"
          github_token: ${{ secrets.GITHUB_TOKEN }}
      - name: Enable auto-merge for FluxCD PRs
        run: gh pr merge --auto --merge "$PR_URL"
        env:
          PR_URL: ${{github.event.pull_request.html_url}}
          GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}
```

`Create Pull Request`のステップで、`main`ブランチへの PR を作成し、`Enable auto-merge for FluxCD PRs`のステップで、PR をマージします。

:::note warn
Flux の[公式ドキュメント](https://fluxcd.io/flux/use-cases/gh-actions-auto-pr/)では GithubActions の発火を

```
on:
  create:
    branches: ['staging']
```

で指定していますが、現在この記法はサポートされていないため注意が必要です。(参考: https://github.com/orgs/community/discussions/26286#discussioncomment-3251208)

:::

この状態で Flux に push をさせると、403 Forbidden エラーが発生しました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/8c562b58-df9f-1cc9-4cfa-b57a24e9ad01.png)

GithubActions に PullRequest を作成・マージする権限が与えられていないのが原因らしいです。

https://zenn.dev/kenghaya/articles/d7f766e5db6437

レポジトリの設定から GithubActions に権限を付与します。

レポジトリの`Settings` > `Actions` > `General`の`Workflow permissions`から `Read and write permissions`と`Allow GitHub Actions to create and approve pull requests`にチェックを入れます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/aeef95c0-cc37-e1ee-ef27-80136cda4916.png)

これで、FluxCD が 差分を push すると、自動で PR が作成・マージされるようになりました。

### Branch Protection Rules の設定

このままだと Kubernetes のテストの結果に関わらずマージがされてしまうので、Branch Protection Rules で、CI が通っていないと PR をマージできないように設定します。

レポジトリの`Settings` > `Branches` > `Branch protection rules`から`main`ブランチの設定を開きます。

`Require status checks to pass before merging`にチェックを入れ、CI のジョブ名を選択します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/b68ae408-c4c6-3416-4108-0de2e41a6dca.png)

これで CI が通らないと PR をマージできないようになりました。

## まとめ

今回は、FluxCD の設定を変更して、アプリケーションの Image が更新されたときに、その変更を適用するような Pull Request を作成するようにしました。

次回は`main`ブランチへマージする前に Staging 環境のクラスタで動作確認できるようにしたいと思います。

## 参考資料

https://fluxcd.io/flux/use-cases/gh-actions-auto-pr/

https://walnuts.hatenablog.com/entry/2024/02/04/021245

https://zenn.dev/kenghaya/articles/d7f766e5db6437
