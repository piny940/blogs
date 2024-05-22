## はじめに

この記事では、Kustomize で namePrefix や nameSuffix を使う際に、Custom Resource への参照に prefix や suffix がついてくれない問題について解説します。

## 環境

```bash
$ kustomize version
v5.4.1
```

## 問題

Kustomize には、`namePrefix` や `nameSuffix` といった機能があります。これを使うことで、リソースの名前に prefix や suffix をつけることができます。

参考: https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/nameprefix/

**File Input**

```yaml
# deployment.yaml
apiVersion: v1
kind: Deployment
metadata:
  name: gd
spec:
  template:
    spec:
      containers:
        - name: gd
          env:
            - name: HOGE
              valueFrom:
                secretKeyRef:
                  name: gs
                  key: username
```

```yaml
# secret.yaml
apiVersion: animal/v1
kind: Secret
metadata:
  name: gs
```

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - secret.yaml
namePrefix: sample-
```

**Build Output**

```yaml
apiVersion: animal/v1
kind: Secret
metadata:
  name: sample-gs
---
apiVersion: v1
kind: Deployment
metadata:
  name: sample-gd
spec:
  template:
    spec:
      containers:
        - env:
            - name: HOGE
              valueFrom:
                secretKeyRef:
                  key: username
                  name: sample-gs # 参照するSecretの名前が変わる
          name: gd
```

上の例で示すように、`Deployment`や`Secret`などの Kubernetes にもともとあるオブジェクトであれば、それを参照する部分の名前も prefix がついてくれます。

しかし、以下の例に示すように、Custom Resource を参照する場合には prefix がつきません。

`kind: Secret`の部分を`kind: MyCustomResource`に変更してみます：

**File Input**

```yaml
# deployment.yaml (変更なし)
apiVersion: v1
kind: Deployment
metadata:
  name: gd
spec:
  template:
    spec:
      containers:
        - name: gd
          env:
            - name: HOGE
              valueFrom:
                secretKeyRef:
                  name: gs
                  key: username
```

```yaml
# secret.yaml
apiVersion: animal/v1
kind: MyCustomResource # <-- Changed!
metadata:
  name: gs
```

```yaml
# kustomization.yaml (変更なし)
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - secret.yaml
namePrefix: sample-
```

**Build Output**

```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: sample-gd
spec:
  template:
    spec:
      containers:
        - env:
            - name: HOGE
              valueFrom:
                secretKeyRef:
                  key: username
                  name: gs # 参照するリソースの名前にPrefixがつかない！
          name: gd
---
apiVersion: animal/v1
kind: MyCustomResource
metadata:
  name: sample-gs
```

すると、`kind: MyCustomResource`を参照する部分の名前には prefix がついてくれません。

このまま`kubectl apply`すると、`could not find the requested resource`のエラーが発生します。

## 解決策

`nameReference`を指定することで、Custom Resource を参照する部分にも prefix をつけることができます。

参考: https://github.com/kubernetes-sigs/kustomize/blob/master/examples/transformerconfigs/README.md#name-reference-transformer

**File Input**

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - secret.yaml
namePrefix: sample-
configurations:
  - namereference.yaml # <-- New!
```

```yaml
# namereference.yaml
nameReference:
  - kind: MyCustomResource # 追加するCustomResourceのkind
    fieldSpecs:
      - kind: Deployment
        path: spec/template/spec/containers/env/valueFrom/secretKeyRef/name # 参照元リソースのpath
```

**Build Output**

```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: sample-gd
spec:
  template:
    spec:
      containers:
        - env:
            - name: HOGE
              valueFrom:
                secretKeyRef:
                  key: username
                  name: sample-gs # 参照するリソースの名前にprefixがつく
          name: gd
---
apiVersion: animal/v1
kind: MyCustomResource
metadata:
  name: sample-gs
```

これで、Custom Resource を参照する部分にも prefix がついた状態でリソースを作成することができます。

## まとめ

今回は CustomResource への参照に prefix がつかない問題について解説しました。`nameReference`を使うことで、Custom Resource を参照する部分にも prefix をつけることができます。

今後はこれを活用して Staging 環境の構築を行っていこうと思います。

## 参考文献

https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/nameprefix/

https://github.com/kubernetes-sigs/kustomize/blob/master/examples/transformerconfigs/README.md#name-reference-transformer
