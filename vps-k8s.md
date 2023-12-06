## はじめに
最近、VPS上で公開していた[ポートフォリオ](https://www.piny940.com)や[歌枠データベース](https://song-list.piny940.com)をDocker上で立てられるようにし、自宅サーバーへと完全移行しました。

(自宅サーバーへサーバーを移行した話はこの辺りの記事にまとめました。)

https://qiita.com/piny940/items/606a6b03c390c136c1f7

https://qiita.com/piny940/items/e41c2709692195f823b5

https://qiita.com/piny940/items/602316d913938db01c81

しかし、自宅サーバーは諸々の理由で頻繁に電源が落ちるため、kubernetesを使って冗長構成にしたいと思うようになりました。また、毎回sshでログインしてデプロイをするのが手間だったので自動デプロイの基盤をkubernetesで作りたいと思うようになり、意を決してkubernetesのお勉強を始めました！

前回はminikube上にNext.js + nginxのサーバーを立てました。

https://qiita.com/piny940/items/0feef2500c8111a520ea

今回は、前々回↓構築したVPS上のkubernetesクラスタ上にNext.js + nginxのサーバーをデプロイしようと思います。

https://qiita.com/piny940/items/3c1c10b80c7e173d527d

完成品はGithub上に公開してありますので参考程度にどうぞ
(`kubernetes/metallb`、`kubernetes/portfolio`が今回扱う対象です)。
https://github.com/piny940/infra/tree/main/kubernetes

## 環境
- lemon: Kagoya Cloud VPS 2コア 2GB
- lime: Kagoya Cloud VPS 2コア 2GB

（lemonやlimeはマシン名です）

- デプロイツール: kubeadm
- CRI: cri-dockerd
- CNI: Weave net

## ingress-nginxをインストール
まずはingress-nginxをインストールします。(私はこれを忘れて数十分悩んだ)

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```

## MetalLBをインストール
次に[MetalLB](https://metallb.universe.tf)をインストールします。
これは何かと言うと、AWS等のクラウドが提供しているk8sサービス(EKSとか)は`type=LoadBalancer`であるServiceに対して外からアクセスする仕組みを提供してくれているのですが、VPSやオンプレミスのサーバーではその仕組みが存在しないため、代わりにサービスを外部に公開する仕組みを提供してくれるものです。
MetalLBについての詳しい説明はこのサイトが分かりやすかったです。
https://blog.framinal.life/entry/2020/04/16/022042

[ドキュメント](https://metallb.universe.tf/installation/)に従ってインストールをしていきます。

```
# see what changes would be made, returns nonzero returncode if different
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

# actually apply the changes, returns nonzero returncode on errors only
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system$ kubectl edit configmap -n kube-system kube-proxy
```

次回以降`kubeadm init`したときにこの設定をしなくてもいいように、`kubeadm-config.yaml`に次のコードを書き足します。
```kubeadm-config.yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```
インストールはkustomizeを使って行いました。
```metallb/kustomization.yaml
namespace: metallb-system

resources:
  - github.com/metallb/metallb/config/native?ref=v0.13.12
```

IPAddressPool, L2Advertisementの設定を記述します。
```metallb/config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
    - 192.168.11.61-192.168.11.70
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
    - default
```

## Next.jsのデプロイ
Next.jsアプリのデプロイです。
```
$ kubectl apply -k .
```

これでデプロイ作業は完了です。serviceを見てみるとingress-nginx-controllerにEXTERNAL-IPが設定されていると思います。

```
kubectl get service -A
NAMESPACE        NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE
default          kubernetes                           ClusterIP      10.96.0.1        <none>          443/TCP                      4d7h
default          {アプリ名}                            ClusterIP      10.106.129.21    <none>          4400/TCP                     8h
ingress-nginx    ingress-nginx-controller             LoadBalancer   10.109.36.122    192.168.11.61   80:30292/TCP,443:31968/TCP   6h22m
ingress-nginx    ingress-nginx-controller-admission   ClusterIP      10.106.213.123   <none>          443/TCP                      6h22m
kube-system      kube-dns                             ClusterIP      10.96.0.10       <none>          53/UDP,53/TCP,9153/TCP       4d7h
metallb-system   webhook-service                      ClusterIP      10.111.253.56    <none>          443/TCP                      3m1s
```

VPS上でEXTERNAL-IPのアドレスにcurlすると確かにnginxのページが返ってきます。
```
$ curl 192.168.11.61
```

## cloudflareで公開
このままだとEXTERNAL-IPに外部からアクセスすることが出来ません。
これがいい方法なのかはわからないのですが、私はCloudflare Tunnelを使って公開することにしました。
Cloudflareのコンソールから`Zero Trust > Access > Tunnels`を開き、「Create a tunnel」からTunnelを作成します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/feb3c582-1293-618d-4e9d-023879490aaf.png)

Subdomainにはingress-nginxで指定したサブドメインを使用し、URLにはServiceのIPとポートを指定します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/de912bbd-bc72-fc5b-7aa9-82b171ba3e96.png)

これでCloudflareに設定したアドレスにアクセスするとNext.jsのページが表示されるようになるはずです！


## 最後に
今回はVPSで立てているkubernetesクラスタ上でNext.js + nginxサーバーを立てて公開する作業を行いました。
次回はいよいよお家kubernetesクラスタ作りに取り掛かりたいと思います。


## 参考資料

https://zenn.dev/vampire_yuta/articles/ccbc57be8e092a

https://blog.framinal.life/entry/2020/04/16/022042
