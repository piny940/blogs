## はじめに

この記事は「お家 Kubernetes 環境を作ろう」シリーズの 1 つです。前回は External Secret OperatorとVaultを連携してkubernetesのsecret基盤を整えました。

https://qiita.com/piny940/items/66214e8f7c8af18ba014

今回は flux notificationを使ってfluxの通知がslackに送られるようにしていきたいと思います。

完成品は Github 上に公開してありますので参考程度にどうぞ
https://github.com/piny940/infra/tree/main/kubernetes

## 環境

OS:  
Ubuntu22.04

kubeadm:

```
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"28", GitVersion:"v1.28.2", GitCommit:"89a4ea3e1e4ddd7f7572286090359983e0387b2f", GitTreeState:"clean", BuildDate:"2023-09-13T09:34:32Z", GoVersion:"go1.20.8", Compiler:"gc", Platform:"linux/amd64"}
```

flux:

```
$ flux version
flux: v2.2.0
distribution: flux-v2.2.0
helm-controller: v0.37.0
image-automation-controller: v0.37.0
image-reflector-controller: v0.31.1
kustomize-controller: v1.2.0
notification-controller: v1.2.2
source-controller: v1.2.2
```

## 前提

- flux の helm controller・kustomize controller が動作している
- External Secretが動作している

## Slackの設定
[Slack API](https://api.slack.com/apps)をでCreate New Appをし、fluxの通知を送るためのSlack Appを作成します。

作成したら`OAuth & Permission`の`Scopes > Bot Token Scopes`からOAuth Scopeを追加します。今回はwebhookを用いてメッセージを送信するため、`incoming-webhook`を追加しました。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/0438fa50-72ed-544e-8f36-9527c9edc5e2.png)

`Install App`からチャンネルにSlack Appをインストールします。インストールしたら`Webhook URL`をコピーしてvault等に置いておきましょう。

## External Secretを作成
外部に配置したSecretを読めるよう、External Secretを定義します。External Secret以外を使っている場合はそれぞれの方法に従ってください。

`flux-alerts/external-secret.yaml`に次のように記述します。

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: slack-webhook-url
  namespace: flux-system
spec:
  secretStoreRef:
    name: vault-secret-store
    kind: ClusterSecretStore
  refreshInterval: 1m
  data:
    - secretKey: address
      remoteRef:
        key: flux
        property: slack-webhook-url
```
`spec.secretStoreRef`にはClusterSecretStore (あるいはSecretStore)の名前を記述します。また、`spec.data[].remoteRef`にはSlackのWebhook URLへのパスを指定します。

## Provider, Alertを作成
`flux-alerts/provider.yaml`に次のように記述します。

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: slack-bot
  namespace: flux-system
spec:
  type: slack
  secretRef:
    name: slack-webhook-url
```

`flux-alerts/slack-alerts.yaml`に次のように記述します。
```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: slack-alert
  namespace: flux-system
spec:
  summary: "Flux alert"
  eventMetadata:
    env: "production"
    cluster: "cluster"
    region: "asia/japan"
  providerRef:
    name: slack-bot
  eventSeverity: info
  eventSources:
    - kind: GitRepository
      name: "*"
    - kind: Kustomization
      name: "*"
    - kind: HelmRepository
      name: "*"
    - kind: HelmRelease
      name: "*"
```

kustomization.yamlにこれらを登録すればfluxの通知がslackに送られるようになります。

## 最後に
今回はflux notificationを用いてfluxの通知がslackに送られるようにしました。
次回はkubernetesクラスタ上でDBサーバーを立てていこうと思います。
