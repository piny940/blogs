## はじめに

前回は external-dns を改良して kubernetes で作成した ingress の DNS 設定が自動でされるようにしました。

https://qiita.com/piny940/items/a151757676beda68e7ae

今回は kubernetes で staging 環境を作っていこうと思います。

完成品は Github 上に公開してありますので参考程度にどうぞ
https://github.com/piny940/infra

## 前提条件

- flux kustomization の環境が整っている
- (flux の AutoImageUpdate 環境が整っている)

## ディレクトリ構成

staging 環境を作る際に最も工夫したのは「production 環境との共通化」です。
今回は以下のように「production 環境をベースとして、staging 環境は production 環境を override する」という形にしました。
`production`の`deployment`や`service`を`staging/kustomization.yaml`から読み込み、必要に応じて`deployment-overlay.yaml`などで上書きするようにしています。

```
app
├── image-policy.yaml
├── external-secret.yaml
├── ingress.yaml
├── kustomization.yaml
├── production
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── staging
    ├── deployment-overlay.yaml
    └── kustomization.yaml
```

なお、次に示すように、production と staging のマニフェストを`base/`を元に作るという方法も考えられましたが、`base`, `production`, `staging`の 3 つのマニフェストを作らないといけないのは手間だったため、今回は`production`を元に`staging`を作成するという選択をしました。この辺りは人それぞれの状況に応じて調整してください。

```
app
├── image-policy.yaml
├── external-secret.yaml
├── ingress.yaml
├── kustomization.yaml
├── base
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── production
│   ├── deployment-overlay.yaml
│   └── kustomization.yaml
└── staging
    ├── deployment-overlay.yaml
    └── kustomization.yaml
```

## マニフェストの内容

トップレベルの`app/kustomization`は次のように記述しました。
トップレベルに配置されている`external-secret`、`image-policy`、`ingress`と`production`、`staging`を読み込んでいます。

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: default
resources:
  - external-secret.yaml
  - image-policy.yaml
  - ingress.yaml
  - ./production
  - ./staging
```

トップレベルの`image-policy`や`ingress`には production と staging の両方の設定を記述しています。  
↓`app/image-policy.yaml`

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: app
spec:
  image: ghcr.io/piny940/app
  interval: 10m
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: app
spec:
  imageRepositoryRef:
    name: app
  policy:
    semver:
      range: 1.0.x
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: app-staging
spec:
  imageRepositoryRef:
    name: app
  filterTags:
    pattern: "^main-[a-z0-9]+-(?P<ts>[0-9]+)$"
    extract: "$ts"
  policy:
    numerical:
      order: asc
```

↓`app/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port:
                  number: 4400
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-staging
                port:
                  number: 4400
```

`production/`ディレクトリの中には通常通りに`deployment`と`service`と`kustomization`を記述します。  
ただし、staging 環境の deployment 等と区別ができるように、`env: production`を `commonLabels` に追加します。
↓`app/production/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: default
commonLabels:
  env: production
resources:
  - deployment.yaml
  - service.yaml
```

`staging/`ディレクトリの`kustomization`からは`production`のマニフェストを見に行き、image や label などを上書きします。
↓`app/staging/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
nameSuffix: -staging
commonLabels:
  env: staging
images:
  - name: ghcr.io/piny940/app
    newName: ghcr.io/piny940/app # {"$imagepolicy": "default:app-staging:name"}
    newTag: main-8dd87cde0b268274a78d350757d7fb88b1e4a62c-2 # {"$imagepolicy": "default:app-staging:tag"}
patches:
  - target:
      kind: Deployment
      name: app
    path: deployment-overlay.yaml
resources:
  - ../production
```

重要な点について解説していきます。  
参考: https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/

- nameSuffix  
  リソースの name の後ろに`-staging`を追加します。
- commonLabels  
  `metadata.labels`や`selector.matchLabels`に`env: staging`を追加します。これにより、他の service や ingress から参照する際に`production`のリソースと`staging`を区別することができるようにしています。
- images  
  production 環境とは別の image を参照するために、images で上書きを行っています。flux の`ImageAutoUpdate`のためのコメントも記述しておくと image の更新を自動でやってくれるようになります。
- patches  
  `kustomization`ファイルから上書きできないプロパティ(`env`など)については patch ファイルを使って直接上書きを行います。

## 最後に

今回は kubernetes 上で staging 環境を作る作業をしました。kustomize の強みを活かして上手く production 環境と共通化ができたのではないかと思っています。(こうすればもっとキレイになる！などがあればぜひ教えてください！)
次回は kubernetes 上で redis 環境を整えていきたいと思います。
