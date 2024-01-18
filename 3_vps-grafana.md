## はじめに
この記事は「お家Kubernetes環境を作ろう」シリーズの1つです。前回はVPS上でNext.js + nginxサーバーを立てて公開するところまで行いました。

https://qiita.com/piny940/items/0ee6a391802c21ab48c1

今回は、VPSで立てたクラスタ上でGrafanaを動かしてみようと思います。

## 環境
- lemon: Kagoya Cloud VPS 2コア 2GB
- lime: Kagoya Cloud VPS 2コア 2GB

（lemonやlimeはマシン名です）

- デプロイツール: kubeadm
- CRI: cri-dockerd
- CNI: Weave net

## Helmをインストール
Helmがまだ入っていなかったので、[ドキュメント](https://helm.sh/docs/intro/install/)に従いインストールしました。

Ubuntu用↓
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## Prometheusをインストール
[helm-charts](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus#configuration)からインストールします。

レポジトリを追加
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

namespaceを作成
```namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prometheus
```

インストール
```
helm install -n prometheus prometheus prometheus-community/prometheus
```

## CSI driverをインストール
まだインストールしていない場合はインストールします。今回はlonghornを使うことにしました。CSI周りはまたの機会にちゃんと調べる。

最初はminikube上で試そうとしたのですが、どうもminikube上では動かなさそうだったので諦めました。(参考: https://github.com/longhorn/longhorn/discussions/2702)


レポジトリを追加
```
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

namespaceを作成
```
kubectl create namespace longhorn-system
```

インストール
```
helm install longhorn longhorn/longhorn --namespace longhorn-system
```

longhorn周りのPodやServiceが正常に起動していることを確認します。

```
$ kubectl get pods -n longhorn-system --watch
NAME                                                READY   STATUS    RESTARTS   AGE
csi-attacher-79b44f5d-f2x2z                         1/1     Running   0          3m7s
csi-attacher-79b44f5d-fxf7v                         1/1     Running   0          3m7s
csi-attacher-79b44f5d-qv4p6                         1/1     Running   0          3m7s
csi-provisioner-c5bb4fff7-d4vx2                     1/1     Running   0          3m7s
csi-provisioner-c5bb4fff7-shhbj                     1/1     Running   0          3m7s
csi-provisioner-c5bb4fff7-smn68                     1/1     Running   0          3m7s
csi-resizer-8cc975c7f-5sxh6                         1/1     Running   0          3m7s
csi-resizer-8cc975c7f-ltpb7                         1/1     Running   0          3m7s
csi-resizer-8cc975c7f-rxbph                         1/1     Running   0          3m7s
csi-snapshotter-58bb8475bc-dq2kn                    1/1     Running   0          3m7s
csi-snapshotter-58bb8475bc-gbbpx                    1/1     Running   0          3m7s
csi-snapshotter-58bb8475bc-w9rrx                    1/1     Running   0          3m7s
engine-image-ei-68f17757-l8zbb                      1/1     Running   0          3m13s
instance-manager-d93c79dee0d0c8a4537d711203fe7489   1/1     Running   0          3m13s
longhorn-csi-plugin-f5w8l                           3/3     Running   0          3m6s
longhorn-driver-deployer-7cc7cc8559-fgpkp           1/1     Running   0          3m51s
longhorn-manager-284sp                              1/1     Running   0          3m51s
longhorn-ui-7c667cb6fc-9pnsd                        1/1     Running   0          3m51s
longhorn-ui-7c667cb6fc-v5mth                        1/1     Running   0          3m51s

$ kubectl get svc -n longhorn-system
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
longhorn-admission-webhook    ClusterIP   10.103.109.251   <none>        9502/TCP   6m1s
longhorn-backend              ClusterIP   10.96.249.117    <none>        9500/TCP   6m1s
longhorn-conversion-webhook   ClusterIP   10.111.32.198    <none>        9501/TCP   6m1s
longhorn-engine-manager       ClusterIP   None             <none>        <none>     6m1s
longhorn-frontend             ClusterIP   10.111.152.92    <none>        80/TCP     6m1s
longhorn-recovery-backend     ClusterIP   10.107.29.92     <none>        9503/TCP   6m1s
longhorn-replica-manager      ClusterIP   None             <none>        <none>     6m1s
```

## Grafanaをインストール
レポジトリを追加。
```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

設定ファイルを記述します。datasourceに先程導入したprometheusを指定しています。また、データの永続化のためにpersistenceを設定し、storageClassNameには先程導入したlonghornを指定します。
```grafana/helm-values.yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: "Prometheus"
        type: prometheus
        access: proxy
        url: http://prometheus-server.prometheus.svc.cluster.local
persistence:
  enabled: true
  storageClassName: longhorn
  size: 1Gi
```

```
helm install -n grafana grafana grafana/grafana -f grafana/helm-values.yaml
```

Grafanaのコンソールにログインするのに必要なパスワードを取得するコマンドが表示されますのでパスワードを取得してメモっておきましょう。

## Ingressをインストール

外部からgrafanaのコンソールにアクセスできるようにするために、ingressを設定します。
ingressには[ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start)を使用します。

ingress-nginxを(初めて使う場合は)インストール
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

設定ファイルを記述します。
```grafana/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: grafana
spec:
  ingressClassName: nginx
  rules:
    - host: grafana.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                  number: 80
```

これでingressが作られ、表示されるIPアドレスにアクセスするとGrafanaのコンソールを見ることが出来ます。
```
$ kubectl get ingress -n grafana
NAME              CLASS   HOSTS                 ADDRESS         PORTS   AGE
grafana-ingress   nginx   grafana.example.com   192.168.11.61   80      4m4s
```

## 最後に
今回はGrafanaを導入してクラスタを監視する基盤を整えました。しかし現時点ではGrafanaのコンソールにログインしてもdashboardに何も設定がされていない状態です。次回はGrafanaのdashboardの設定を進めていこと思います。

## 参考資料

https://helm.sh/ja/docs/

https://prometheus.io

https://qiita.com/showchan33/items/dc57030f8794ca4a7c31

https://hogetech.info/oss/prometheus

https://qiita.com/yurak/items/c3d098c4596aaf6ac674

https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus#readme

https://light-of-moe.ddo.jp/~sakura/diary/?p=1575

https://kubernetes.io/ja/docs/concepts/storage/persistent-volumes

https://qiita.com/MetricFire/items/cc9fe9741288048f4588
