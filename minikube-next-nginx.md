## はじめに
この記事は「お家Kubernetes環境を作ろう」シリーズの1つです。前回はマルチノードクラスタ上でpodを立てるところまでを行いました。

https://qiita.com/piny940/items/3c1c10b80c7e173d527d

今回は手元のminikube上にNext.js + Nginxのサーバーを立てていこうと思います。

完成品はGithub上に公開してありますので参考程度にどうぞ
https://github.com/piny940/infra/tree/main/kubernetes/portfolio


## 環境
- Ubuntu22.04
- minikube 1.32.0
- Docker 24.0.7
- Kubernetes v1.28.3

## 前提条件
- 動くDockerfileが用意できている
- dockerhubのアカウントを持っている

## Dockerイメージをpush
まず、[dockerhub](https://hub.docker.com)でレポジトリを作成します。(このレポジトリ名は後で使います)

次に`docker build`でdockerイメージを作成します。
```
$ docker build .
```
作成されたイメージのIDを確認します。
```
$ docker images
```

タグをつけます。`tagname`はバージョン管理で使用するため、`yyyyMMddHHmmss`の形にしました。

```
$ docker tag {imageID} {アカウント名}/{レポジトリ名}:{タグ名}
```

pushします。
```
$ docker push {アカウント名}/{レポジトリ名}:{タグ名}
```

## minikubeを起動
```
minikube start --driver=docker
```
driverはdockerを使用することにしました。デフォルトのQEMUでは何だったか忘れましたが何かに対応してなくてできませんでした。

## deployment.yamlを作成
```deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {アプリ名}
  labels:
    app: {アプリ名}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {アプリ名}
  template:
    metadata:
      labels:
        app: {アプリ名}
    spec:
      containers:
        - name: app
          image: {イメージ名}
          ports:
            - containerPort: {サーバーのポート番号}
```

イメージ名は`{dockerhubのアカウント名}/{レポジトリ名}:{タグ名}`の形で書きます。

これで`kubectl apply -f deployment.yaml`とするとdeploymentとpodが作成されます。
```
$ kubectl get deployment
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
{アプリ名}   1/1     1            1           45h
```

## service.yamlを作成
```service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {アプリ名}
spec:
  selector:
    app: {アプリ名}
  ports:
    - name: http
      protocol: TCP
      port: {サーバーのポート番号}
      targetPort: {サーバーのポート番号}
  type: ClusterIP
```
typeはClusterIPを指定します。serviceのtypeについてはこの記事がわかりやすかったです。
https://www.ios-net.co.jp/blog/20230621-1179/

正常に作成されていることを確認します。
```
$ kubectl get svc
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes          ClusterIP   10.96.0.1      <none>        443/TCP          5d21h
{アプリ名}   ClusterIP   10.97.105.94   <none>        4400/TCP         45h
```

## ingress-nginx.yamlを作成
kubernetes上にnginxコンテナを作成してserviceに対してプロキシしたい場合は、[ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/)が便利です。

導入は概ねチュートリアルそのままで出来ます。
https://kubernetes.io/ja/docs/tasks/access-application-cluster/ingress-minikube/

まずはminikubeでingressを使えるようにします。
```
$ minikube addons enable ingress
```

次に`ingress-nginx.yaml`を作成し、次のように記述します。
```ingress-nginx.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {アプリ名}
spec:
  ingressClassName: nginx
  rules:
    - host: hello-world.info
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {アプリ名}
                port:
                  number: {サーバーのポート番号}
```

`kubectl apply -f ingress-nginx.yaml`でingressが立ち上がるはずです。

```
$ kubectl get ingress
NAME        CLASS   HOSTS              ADDRESS        PORTS   AGE
{アプリ名}   nginx   hello-world.info   192.168.49.2   80      69s
```
これでアクセスできるように…と言いたいところなのですが、driverをdockerにしてminikubeを起動するとなぜか上手く動作しないらしいです。
そのため、代替手段として次のコマンドを使用します。
```
$ minikube service ingress-nginx-controller --url -n ingress-nginx
http://127.0.0.1:44897
http://127.0.0.1:43161
❗  Docker ドライバーを linux 上で使用しているため、実行するにはターミナルを開く必要があります。
```
これで表示されたアドレスにアクセスするとnginxの404が返ってくるはずです。

nextのアプリにアクセスするには次のようにホスト名を指定します。
```
$ curl {表示されたIPアドレス} -H 'Host: hello-world.info'
```

## kustomization.yamlを作成
前章でNext + nginxのサーバーは立てられたのですが、マニフェストファイルがいくつもあると管理が大変になるため、kustomizeを導入することにします。
```kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: default
resources:
  - deployment.yaml
  - service.yaml
  - ingress-nginx.yaml
```
これで環境が変わっても`kubectl apply -k .`とするだけでマニフェストの適用が完了します。

## 最後に
今回はminikube上でNextサーバーをnginxを通してアクセスできるようにしました。次回は今回作ったものをVPS上で再現していきたいと思います。
