## はじめに
この記事は「お家Kubernetes環境を作ろう」シリーズの1つです。前回はGithub Actionsを用いてDockerイメージを自動でビルドできるようにしました。

https://qiita.com/piny940/items/4f4158b889db19418588

今回はいよいよ自宅サーバーでk8sクラスタを立てていこうと思います。

完成品はGithub上に公開してありますので参考程度にどうぞ
https://github.com/piny940/infra/tree/main/kubernetes/portfolio

## 環境
Ubuntu22.04
AMD® Ryzen 5 5600 6-core processor × 12
```
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"28", GitVersion:"v1.28.2", GitCommit:"89a4ea3e1e4ddd7f7572286090359983e0387b2f", GitTreeState:"clean", BuildDate:"2023-09-13T09:34:32Z", GoVersion:"go1.20.8", Compiler:"gc", Platform:"linux/amd64"}

$ helm version
version.BuildInfo{Version:"v3.13.1", GitCommit:"3547a4b5bf5edb5478ce352e18858d8a552a4110", GitTreeState:"clean", GoVersion:"go1.20.8"}

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

## コンテナランタイムのインストール
コンテナランタイムにはcontainerdを使うことにしました。Ubuntuの場合はdockerをインストールするとcontainerdが自動でついてくるため、dockerをインストールすることでcontainerdをインストールします。
参考: https://docs.docker.com/engine/install/ubuntu/

repositoryを追加
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
インストール
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## containerdの設定
[ドキュメント](https://kubernetes.io/ja/docs/setup/production-environment/container-runtimes/#containerd)に従って設定を行います。
- `containerd config default > /etc/containerd/config.toml`でcontainerdの設定をリセット
- `/etc/containerd/config.toml`からSystemdCgroupの設定を見つけて`SystemdCgroup = true`とする。
- `containerd`をrestart

## kubeadm init
swapを無効化します。
```
sudo swapoff -a
```

`kubeadm-config.yaml`に次のように記述します。
```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true # MetalLBに必要
```

ここで問題が発生しました。CNI(Weave)をインストールした後、flux bootstrapをしたのですが、PODがすべてPendingになっていました。
```
$ kubectl get pods -A
NAMESPACE     NAME                                           READY   STATUS    RESTARTS   AGE
flux-system   helm-controller-5f964c6579-ptjqh               0/1     Pending   0          29m
flux-system   image-automation-controller-7764f8957c-j72rk   0/1     Pending   0          29m
flux-system   image-reflector-controller-84c449dc57-xhffc    0/1     Pending   0          29m
flux-system   kustomize-controller-9c588946c-f92hb           0/1     Pending   0          29m
flux-system   notification-controller-76dc5d768-hksls        0/1     Pending   0          29m
flux-system   source-controller-6c49485888-pwxpd             0/1     Pending   0          29m
kube-system   coredns-5dd5756b68-6wzmj                       1/1     Running   0          32m
kube-system   coredns-5dd5756b68-cphml                       1/1     Running   0          32m
kube-system   etcd-cherry                                    1/1     Running   530        32m
kube-system   kube-apiserver-cherry                          1/1     Running   529        32m
kube-system   kube-controller-manager-cherry                 1/1     Running   538        32m
kube-system   kube-proxy-x2pgh                               1/1     Running   0          32m
kube-system   kube-scheduler-cherry                          1/1     Running   529        32m
kube-system   weave-net-n5rmh                                2/2     Running   0          30m
```
ノードをdescribeしてeventを確認
```
$ kubectl describe node node1
...
 Warning  FailedScheduling  105s (x7 over 32m)  default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```
今回はノードは自宅サーバー1台のため、ノードのtaintを削除する必要があるらしい。
```
$ kubectl describe node cherry
Name:               cherry
...
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
...
```
このTaintがあると、ノードがワーカーノードとして働かなくなるようなので、taintを削除します。
```
$ kubectl taint nodes node1 node-role.kubernetes.io/control-plane-
```

ここまでやったが、POD内から外部のネットワークに接続する際のDNSが死んでいることを確認しました。

<details><summary>やったこと</summary>
POD内からcurlするのには https://github.com/nicolaka/netshoot を使った。

```
$ kubectl debug podname -it --image=nicolaka/netshoot
```
```
$ dig github.com
;; communications error to 10.96.0.10#53: timed out
;; communications error to 10.96.0.10#53: timed out
;; communications error to 10.96.0.10#53: timed out

; <<>> DiG 9.18.13 <<>> github.com
;; global options: +cmd
;; no servers could be reached
```
一方、DNSを指定して実行すると成功
```
$ dig github.com @8.8.8.8

; <<>> DiG 9.18.13 <<>> github.com @8.8.8.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 30078
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;github.com.			IN	A

;; ANSWER SECTION:
github.com.		60	IN	A	20.27.177.113

;; Query time: 16 msec
;; SERVER: 8.8.8.8#53(8.8.8.8) (UDP)
;; WHEN: Thu Dec 14 14:10:59 UTC 2023
;; MSG SIZE  rcvd: 55
```
また、corednsのPODの中であればDNSが正常に機能した。
これらのことから、POD間のネットワークが上手く動いていないのだと結論づけた。

</details>

色々試しましたが、結論として、weaveはメンテナンスがされなくなって久しいため、flannelを使うことにしました。

## flannelのインストール
まずはkubeadm reset。
`/etc/cni/net.d`は自動では消えないため手動で削除します。
```
$ sudo rm -rf /etc/cni/net.d
```
```
$ kubeadm init ---config=kubeadm-config.yaml
```
```
$ kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
flannelのPodで次のようなエラーが起きました。
```
$ kubectl logs kube-flannel-ds-7jf94 -n kube-flannel
...
E1217 02:43:42.771018       1 main.go:333] Error registering network: failed to acquire lease: node "cherry" pod cidr not assigned
```
CIDRが設定されていないということだったので、`kubeadm-config.yaml`に次のように記述しました。


```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: stable
networking:
  podSubnet: "10.244.0.0/16"
```
するとエラーが消え、正常に動くようになりました。
また、iptablesを確認したところ、ufwのファイアウォールが悪さをしていたため、`ufw disable`で無効化しました。

## flux bootstrap
最後に、flux bootstrapを実行します。
```bash
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --token-auth \
  --owner=piny940 \
  --repository=infra \
  --branch=main \
  --path=kubernetes/flux \
  --read-write-key \
  --personal
```
これで定義したdeploymentが正常に立てられました。

## 最後に
今回はVPS上で立てたクラスタを自宅サーバー上で動かす作業をしました。ノードが1つになったことで生じるエラーや、ファイアウォールのせいで生じるエラーなど、原因をつきとめるのが難しい箇所がいくつかあり苦戦しました。
次回はexternal secretが使えるように基盤を整えたいと思います。

## 参考資料

https://qiita.com/yasuoohno/items/d4f40f343549ca5c295e