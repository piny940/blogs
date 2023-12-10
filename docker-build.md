## はじめに
この記事は「お家kubernetes環境を作ろう」シリーズの1つです。前回はfluxを使ってhelmやkustomizeを自動で適用できるようにしました。

https://qiita.com/piny940/items/e99fea7fcba720047b11

今回は、前々回導入した自動デプロイ機能に関連して、自動ビルド基盤を作っていこうと思います。

完成品はこちらのリポジトリで公開しています。
https://github.com/piny940/portfolio

## 権限の設定
レポジトリの設定から、workflowに権限を与えます。
![workflow permission](https://user-images.githubusercontent.com/1736354/187890839-2f26ce10-2e20-4d7e-ab6e-311c898fc416.png)
![actions permissions](https://user-images.githubusercontent.com/9700541/187891526-5938feb5-d380-4574-a81a-9b621779dead.png)

参考: https://github.com/docker/build-push-action/issues/687

## workflowを設定
`.github/workflows/release.yaml`に次のように記述
```yaml
name: Docker Build for Release
on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  frontend:
    runs-on: ubuntu-latest
    steps:
      - name: Check out
        uses: actions/checkout@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: Githubアカウント名
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Build and push Docker images
        uses: docker/build-push-action@v5
        with:
          push: true
          context: .
          file: Dockerfile
          tags: |
            ghcr.io/Githubアカウント名/アプリ名:v1.0.0-${{ github.sha }}-${{ github.run_number }}
            ghcr.io/Githubアカウント名/アプリ名:latest
```
2~5行目: mainブランチにpushされたときにworkflowを実行します。
6行目: 手動でフローを実行できるようにします。
12~17行目: ghcr.ioにログインします。
20~28行目: DockerイメージをBuild・Pushします。

これでmainブランチにpushしたら自動でDockerイメージがビルドされるようになります。作成されたImageはGithubの`Profile > Packages`から確認できます。

## 最後に
今回はGithub Actionsを用いて自動ビルドの基盤を整えました。次回はStaging環境の作成に挑戦したいと思います。


## 参考資料

https://docs.docker.com/build/ci/github-actions/

https://qiita.com/chroju/items/d98ea757f906a9b5d683

https://github.com/docker/build-push-action/issues/687
