https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide

```bash
$ export SA_SECRET_NAME=$(kubectl get secrets -n vault --output=json | jq -r '.items[].metadata | select(.name|startswith("vault-auth-")).name')
```

```bash
$ export SA_JWT_TOKEN=$(kubectl get secret -n vault $SA_SECRET_NAME --output 'go-template={{ .data.token }}' | base64 --decode)
```

```bash
$ export SA_CA_CRT=$(kubectl config view --raw --minify --flatten \
    --output 'jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
```

```bash
$ export K8S_HOST=$(kubectl config view --raw --minify --flatten \
    --output 'jsonpath={.clusters[].cluster.server}')
```

## 参考資料

https://speakerdeck.com/jacopen/k8stovaultwozu-mihe-wasetesikuretutowomotutosekiyuani

https://developer.hashicorp.com

https://qiita.com/ryysud/items/ec8de49aa39a2f9fbceb

https://qiita.com/ipppppei/items/4aab7005c069bb5f18f5
