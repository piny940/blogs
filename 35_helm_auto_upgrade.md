## はじめに

私が運用している Kubernetes クラスタでは、公開されている Helm Chart を使っていくつかのサービスを立てています。

しかし、これらすべての Chart に対して、定期的にアップデートが来ていないかを確認して手動で設定を更新するのは非常に手間です。

そこで今回は、Flux の公式が紹介している[Helm Promotion](https://fluxcd.io/flux/use-cases/gh-actions-helm-promotion/)の方法に従って、新しいバージョンの Helm Chart がリリースされたら「ステージング環境で動作確認をして」本番環境でも自動でバージョンを更新する仕組みを整えていこうと思います。

## 実装

実装しないといけないのは主に次の 4 つです。

- ステージング環境の Helm
- 本番環境の Helm
- Flux Alert
- GitHub Action

### ステージング環境の Helm

ステージング環境では、マイナーバージョンのアップグレードが勝手に行われるよう、バージョン指定のマイナーバージョンを x で指定しておきます。

```yaml
# staging.helm.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: release-name
spec:
  chart:
    spec:
      version: 19.x
  test:
    enable: true
```

### 本番環境の Helm

本番環境の Helm では、通常通りマイナーバージョンまで指定しておきます。

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: release-name
spec:
  chart:
    spec:
      version: 19.0.1
```

### Flux Alert

Staging 環境で Helm に変更が加えられたら、GitHub Actions をトリガーするように設定します。
Staging 環境で次のように Flux の Provider, Alert を設定します。

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: githubdispatch
  namespace: flux-system
spec:
  type: githubdispatch
  address: <repo address>
  secretRef:
    name: github-token
---
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: helm-promotion
  namespace: flux-system
spec:
  providerRef:
    name: githubdispatch
  summary: "Trigger promotion"
  eventMetadata:
    env: staging
  eventSeverity: info
  inclusionList:
    - ".*.test.*succeeded.*"
  eventSources:
    - kind: HelmRelease
      name: "*"
      namespace: monitoring
    - kind: HelmRelease
      name: "*"
      namespace: prometheus
    - kind: HelmRelease
      name: "*"
      namespace: vault
```

全 namespace に対して指定をしないといけないのは面倒ですが、仕方ないですね
Flux の issue を見てみたら、一応改善する意思はあるみたいなので今後のリリースを待ちましょう

### Github Action

最後に、Flux が GitHub Action をトリガーしたら、本番環境の Helm のバージョンが更新されるように GitHub Action を作成します。

```yaml
name: helm-promotion
on:
  repository_dispatch:
    types:
      - HelmRelease/*
permissions:
  contents: write
  pull-requests: write
jobs:
  promote:
    runs-on: ubuntu-latest
    if: |
      github.event.client_payload.metadata.env == 'staging' &&
      github.event.client_payload.severity == 'info'
    steps:
      - uses: actions/checkout@v4
        with:
          ref: main
      # Parse the event metadata to determine the chart version deployed on staging.
      - name: Get chart info from staging
        id: staging
        run: |
          NAME=${{ github.event.client_payload.involvedObject.name }}
          VERSION=$(echo ${{ github.event.client_payload.metadata.revision }} | cut -d '@' -f1)
          echo NAME=${NAME} >> $GITHUB_OUTPUT
          echo VERSION=${VERSION} >> $GITHUB_OUTPUT
      # Patch the chart version in the production Helm release manifest.
      - name: Set chart version in production
        id: production
        env:
          CHART_VERSION: ${{ steps.staging.outputs.version }}
        run: |
          echo "set chart version to ${CHART_VERSION}"
          FILE_PATH=$(find ./kubernetes/apps/*/production/*helm*.yaml -type f -print | xargs grep "name: ${{ steps.staging.outputs.name }}$" | cut -d':' -f1)
          yq eval '.spec.chart.spec.version=env(CHART_VERSION)' -i $FILE_PATH
      # Open a Pull Request if an upgraded is needed in production.
      - name: Open promotion PR
        uses: peter-evans/create-pull-request@v6
        id: open_pr
        with:
          branch: fluxcd/helm-promotion/${{ steps.staging.outputs.name }}/${{ steps.staging.outputs.version}}
          delete-branch: true
          token: ${{ secrets.GITHUB_TOKEN }}
          commit-message: Update ${{ steps.staging.outputs.name }} to v${{ steps.staging.outputs.version }}
          title: Promote ${{ steps.staging.outputs.name }} release to v${{ steps.staging.outputs.version }}
          body: |
            Promote ${{ steps.staging.outputs.name }} release on production to v${{ steps.staging.outputs.version }}
      - name: Enable auto-merge for FluxCD PRs
        run: gh pr merge --auto --merge "$PR_URL"
        env:
          PR_URL: ${{ steps.open_pr.outputs.pull-request-url }}
          GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}
```

以下の部分で、該当する Helm ファイルを探し出しています。

```
FILE_PATH=$(find ./kubernetes/apps/*/production/*helm*.yaml -type f -print | xargs grep "name: ${{ steps.staging.outputs.name }}$" | cut -d':' -f1)
```

これで Helm の新しいバージョンがリリースされたら自動でステージング・プロダクションに更新が入るようになりました！

## まとめ

今回は Flux の Alert を使って Helm の自動アップグレードを実現しました。ステージング環境を使って新しいバージョンの Helm の動作確認をした上で Production 環境に自動で反映させる、という一連の流れを実装することができて満足しています。

## 参考資料

https://fluxcd.io/flux/use-cases/gh-actions-helm-promotion/
