## はじめに

この記事は「お家 Kubernetes 環境を作ろう」シリーズの 1 つです。前回は 自宅サーバーで k8s クラスタを立てました。

https://qiita.com/piny940/items/699120125d3701eea662

今回は kubernetes の ExternalSecret と vault を連携して Secret 管理の基盤を作っていこうと思います。

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

helm:

```
$ helm version
version.BuildInfo{Version:"v3.13.1", GitCommit:"3547a4b5bf5edb5478ce352e18858d8a552a4110", GitTreeState:"clean", GoVersion:"go1.20.8"}
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
- ingress-nginx が動作している

## vault のインストール

[公式ドキュメント](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-minikube-raft)に従って kubernetes に vault をインストールしていきます。

公式では helm コマンドを用いてインストールしていきますが、ここでは flux の helm controller を使ってインストールします。

`vault/helm.yaml`に次のように記述します。

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: hashicorp
  namespace: vault
spec:
  interval: 1h
  url: https://helm.releases.hashicorp.com
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: vault
  namespace: vault
spec:
  interval: 1h
  chart:
    spec:
      chart: vault
      sourceRef:
        kind: HelmRepository
        name: hashicorp
  values:
    server:
      ingress:
        enabled: true
        ingressClassName: nginx
        pathType: Prefix
        hosts:
          - host: vault.example.com
```

最後の行の`spec.values/server/ingress/hosts[].host`にはご自身の vault 用のホストアドレスを記述してください。
なお、公式ドキュメントでは vault サーバー 3 台の HA 構成となっていますが、今回はメモリ節約のため 1 台で構成していきます。

## External Secret Operator のインストール

[公式ドキュメント](https://external-secrets.io/latest/introduction/getting-started/)を参考にインストールしていきます。

`external-secrets/helm.yaml`に次のように記述します。

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: external-secrets
  namespace: external-secrets
spec:
  interval: 1h
  url: https://charts.external-secrets.io
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: external-secrets
  namespace: external-secrets
spec:
  interval: 1h
  chart:
    spec:
      chart: external-secrets
      sourceRef:
        kind: HelmRepository
        name: external-secrets
```

## vault のセットアップ

まずは vault の初期処理をします。

```bash
$ kubectl exec vault-0 -n vault -- vault operator init \
    -key-shares=1 \
    -key-threshold=1 \
    -format=json > cluster-keys.json
```

`VAULT_UNSEAL_KEY`に unseal 用の鍵を入れます。

```bash
$ VAULT_UNSEAL_KEY=$(jq -r ".unseal_keys_b64[]" cluster-keys.json)
```

unseal します。

```bash
$ kubectl exec vault-0 -n vault -- vault operator unseal $VAULT_UNSEAL_KEY
```

これで vault が使えるようになります。

## vault CLI のインストール

`vault`に CLI で鍵の追加等を行えるように、vault の CLI をインストールします。  
参考: https://developer.hashicorp.com/vault/tutorials/getting-started/getting-started-install

```bash
sudo apt update && sudo apt install gpg wget

wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

gpg --no-default-keyring --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg --fingerprint

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install vault
```

`vault version`で正常にインストールできたことを確認します。

```bash
$ vault version
Vault v1.15.4 (9b61934559ba31150860e618cf18e816cbddc630), built 2023-12-04T17:45:28Z
```

vault にログインするために、`root_token`を確認します。

```bash
$ jq -r ".root_token" cluster-keys.json
```

また、`VAULT_ADDR`には vault を helm でインストールする際に指定した vault のホストアドレスを指定します。

```bash
$ export VAULT_ADDR=https://vault.example.com
```

`vault login`でログインをします。ログイントークンを求められるので先程確認した`root_token`を入力してください。

```bash
$ vault login
```

## vault で kubernetes との接続を許可

```bash
vault auth enable kubernetes
export SA_SECRET_NAME=$(kubectl get secrets --output=json \
    | jq -r '.items[].metadata | select(.name|startswith("vault-auth-")).name')
export SA_JWT_TOKEN=$(kubectl get secret $SA_SECRET_NAME \
    --output 'go-template={{ .data.token }}' | base64 --decode)
export SA_CA_CRT=$(kubectl config view --raw --minify --flatten \
    --output 'jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
export K8S_HOST=$(kubectl config view --raw --minify --flatten \
    --output 'jsonpath={.clusters[].cluster.server}')
vault write auth/kubernetes/config \
     token_reviewer_jwt="$SA_JWT_TOKEN" \
     kubernetes_host="$K8S_HOST" \
     kubernetes_ca_cert="$SA_CA_CRT" \
     issuer="https://kubernetes.default.svc.cluster.local"
```

## Policy, Role の設定

まず、vault の Policy と Role を作成します。  
vault の管理画面(`vault.example.com`)にアクセスし、ログインします。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/143ddd69-5a33-58d9-a42c-99df8ddcea7b.png)

ログインしたら、`Policies > ACL Policies`から Policy を追加します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/6346d47a-f9f7-3631-2711-7b366d2181ed.png)

Name は`k8s-cluster`、Policy は

```
path "k8s/*" {
    capabilities = ["read", "list"]
}
```

と記述します。

次に`Access > Authentication Methods > kubernetes > Create role`からロールを作成します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/9914236b-1322-f96e-d3e6-c498635ab1f2.png)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/b218456e-8990-f895-214d-3a98aa6bd755.png)

Name は`k8s-cluster`、Bound service account names は`vault-auth`、Bound service account namespaces は`vault`、Generated Token's Policies は`k8s-cluster` (先ほど作成した Policy の名前)、Generated Token's Initial TTL は`86400` (24 時間)とします。

最後に、`Secrets Engines > Enable new engine`から`KV`を選択して作成します。Path は`k8s`とします。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/241cd2bc-0572-803f-4ffc-067ad591a7b2.png)

今回は実験のために`k8s/portfolio`に secret を作成しておきます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/da4916f9-7587-0e3d-bf49-3882dfffeda2.png)

## kubernetes 側の設定

まず、kubernetes 側で ServiceAccount と Secret を作成します。  
`vault/service-account.yaml`に次のように記述します。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth
  namespace: vault
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-role-binding
  namespace: vault
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: vault-auth
    namespace: vault
```

`vault/secret.yaml`に次のように記述します。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth-secret
  annotations:
    kubernetes.io/service-account.name: vault-auth
type: kubernetes.io/service-account-token
```

`external-secrets-store/secret-store.yaml`に次のように記述します。

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-secret-store
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: k8s
      version: v2
      namespace: vault
      auth:
        kubernetes:
          mountPath: kubernetes
          role: k8s-cluster
          serviceAccountRef:
            name: vault-auth
            namespace: vault
```

今回は簡単のため`SecretStore`ではなく`ClusterSecretStore`を使うことにしました。

最後に、`app/external-secret.yaml`に次のように記述します。

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: portfolio-secret
spec:
  secretStoreRef:
    name: vault-secret-store
    kind: ClusterSecretStore
  refreshInterval: 1m
  data:
    - secretKey: fuga
      remoteRef:
        key: portfolio
        property: password
```

これで External Secret から key を取得できるようになりました。

## 最後に

今回は External Secret Operator と Vault を連携して kubernetes の secret 基盤を作成しました。次回は今回使った secret 基盤を活用してデータベースサーバーを作っていこうと思います。

## 参考資料

https://qiita.com/ipppppei/items/4aab7005c069bb5f18f5

https://developer.hashicorp.com/vault/docs/platform/k8s
