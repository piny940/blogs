## エラー内容

FluxCD で`flux bootstrap`を実行したところ、`context deadline exceeded`のエラーが発生しました。

```
$ flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --token-auth \
  --owner=piny940 \
  --repository=infra \
  --branch=main \
  --path=kubernetes/_flux/${K8S_ENV} \
  --read-write-key \
  --personal

► connecting to github.com
► cloning branch "main" from Git repository "https://github.com/piny940/infra.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ component manifests are up to date
► installing components in "flux-system" namespace
✔ installed components
✔ reconciled components
► determining if source secret "flux-system/flux-system" exists
► generating source secret
► applying source secret "flux-system/flux-system"
✔ reconciled source secret
► generating sync manifests
✔ generated sync manifests
✔ sync manifests are up to date
► applying sync manifests
✔ reconciled sync configuration
◎ waiting for GitRepository "flux-system/flux-system" to be reconciled
✗ gitrepository 'flux-system/flux-system' not ready: 'failed to checkout and determine revision: unable to clone 'https://github.com/piny940/infra.git': Get "https://github.com/piny940/infra.git/info/refs?service=git-upload-pack": dial tcp: lookup github.com: i/o timeout'
◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
✗ client rate limiter Wait returned an error: context deadline exceeded
► confirming components are healthy
✔ helm-controller: deployment ready
✔ image-automation-controller: deployment ready
✔ image-reflector-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
✗ bootstrap failed with 2 health check failure(s): [error while waiting for GitRepository to be ready: 'gitrepository 'flux-system/flux-system' not ready: 'failed to checkout and determine revision: unable to clone 'https://github.com/piny940/infra.git': Get "https://github.com/piny940/infra.git/info/refs?service=git-upload-pack": dial tcp: lookup github.com: i/o timeout'', error while waiting for Kustomization to be ready: 'client rate limiter Wait returned an error: context deadline exceeded']
```

CNI 周りかな？と思ったのですが、Pod 自体は正常に起動しているようでした。

```
k get po -A
NAMESPACE      NAME                                          READY   STATUS    RESTARTS      AGE
flux-system    helm-controller-76dff45854-mz9rm              1/1     Running   2 (21m ago)   153m
flux-system    image-automation-controller-fb6c9df74-5vk8t   1/1     Running   2 (21m ago)   153m
flux-system    image-reflector-controller-565565d549-zf2mg   1/1     Running   2 (21m ago)   153m
flux-system    kustomize-controller-6bc5d5b96-bl6ct          1/1     Running   2 (21m ago)   153m
flux-system    notification-controller-7f5cd7fdb8-8m9rl      1/1     Running   2 (21m ago)   153m
flux-system    source-controller-54c89dcbf6-qwmzf            1/1     Running   2 (21m ago)   153m
kube-flannel   kube-flannel-ds-7thdj                         1/1     Running   2 (21m ago)   160m
kube-system    coredns-7db6d8ff4d-54hlx                      1/1     Running   2 (21m ago)   162m
kube-system    coredns-7db6d8ff4d-brpgt                      1/1     Running   2 (21m ago)   162m
kube-system    etcd-cherry                                   1/1     Running   3 (21m ago)   163m
kube-system    kube-apiserver-cherry                         1/1     Running   3 (21m ago)   163m
kube-system    kube-controller-manager-cherry                1/1     Running   3 (21m ago)   163m
kube-system    kube-proxy-qlzgx                              1/1     Running   3 (21m ago)   162m
kube-system    kube-scheduler-cherry                         1/1     Running   3 (21m ago)   163m
```

## 解決策

ファイアウォールの問題でした。

`ufw disable`して再起動したら、正常に動作しました。

:::note info
ファイアウォールが無効なままなのはさすがにまずいので適切に設定し直します
:::
