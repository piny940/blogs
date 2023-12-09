## はじめに
この記事は「お家kubernetes環境を作ろう」シリーズの1つです。前回はfluxを使って自動デプロイ基盤を整えました。

https://qiita.com/piny940/items/536123b1c4b1884180fe

今回は、前回導入したfluxを用いてhelmやkustomizeを自動で適用できるようにしていこうと思います。

完成品はこちらのリポジトリで公開しています。
https://github.com/piny940/infra

## 環境
lemonおよびlimeはマシン名です。VPS2台のクラスタで動かしています。
- lemon: Kagoya Cloud VPS 2コア 2GB
- lime: Kagoya Cloud VPS 2コア 2GB

- デプロイツール: kubeadm
- CRI: cri-dockerd
- CNI: Weave Net

## 前提条件
- kubernetesクラスタが動作している
- fluxがインストールされている

## kustomizationの設定
`flux/kustomizations/hello-node`に次のように記述します。
```flux/kustomizations/hello-node
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: hello-node
  namespace: flux-system
spec:
  interval: 10m0s
  path: "./kubernetes/hello-node"
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
```
`spec.path`はアプリの`kustomization.yaml`が置かれているディレクトリを指しています。

この変更をmainにpushするとアプリの`kustomization.yaml`が自動で適用されます。

## Helm Releaseの設定
今回は[ingress-nginx](https://github.com/kubernetes/ingress-nginx)をインストールするための設定を例に示します。

`ingress-nginx/helm.yaml`:
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: ingress-nginx
spec:
  interval: 1h
  url: https://kubernetes.github.io/ingress-nginx
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: ingress-nginx
spec:
  interval: 1h
  chart:
    spec:
      chart: ingress-nginx
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
```

HelmRepositoryではingress-nginxのchartレポジトリを指定し、HelmReleaseではリリースを定義しています。

`ingress-nginx/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: ingress-nginx
resources:
  - helm.yaml
```

`flux/kustomizations/ingress-nginx.yaml`:
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: ingress-nginx
  namespace: flux-system
spec:
  interval: 10m0s
  path: "./kubernetes/ingress-nginx"
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
```

これらの変更をmainにpushすると、ingress-nginxがインストールされます。

## 最後に
今回はfluxを用いてhelmやkustomizeを自動で適用できるようにしました。次回はDockerイメージのpushも自動でできるよう基盤を整えていきたいと思います。

## 参考資料

https://fluxcd.io/

https://qiita.com/ipppppei/items/2c79c9016c7b848762c9
