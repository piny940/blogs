## はじめに
前回はfluxの通知がslackに流れるように設定を行いました。

https://qiita.com/piny940/items/88dedb16585671b6352b

今回は自己署名証明書を用いてingressのTLS設定を行なっていこうと思います。

## 手順

秘密鍵を作成
```
$ openssl genpkey -algorithm ed25519 -out priv.key
```

自己署名証明書を作成
```
$ openssl req -x509 -days 3650 -new -key priv.key -out server.crt
```

```
kubectl create secret tls cluster-tls --key priv.key --cert server.crt
```

Ingressの`spec.tls`を次のように記述
```
...
kind: Ingress
spec:
  tls:
    - hosts:
        - www.example.com
      secretName: cluster-tls
...
```

これで自己署名証明書を用いたTLS通信ができるようになります。

## 参考資料

https://qiita.com/cffnpwr/items/09995c5b19b01d7f1092

https://zenn.dev/vampire_yuta/articles/30bda4332f78c5

https://pebble8888.hatenablog.com/entry/2019/04/30/211832

https://www.openssl.org/docs/man3.0/man1/openssl-genpkey.html

https://www.openssl.org/docs/man3.0/man1/openssl-req.html

https://stackoverflow.com/questions/75358381/failed-to-create-secret-post-http-localhost8080-api-v1-namespaces-keycloak-t

https://stackoverflow.com/questions/46360361/invalid-x509-certificate-for-kubernetes-master

https://developers.cloudflare.com/ssl/origin-configuration/origin-ca

https://a.nel.cloudflare.com/report/v3?s=x%2FfpYTzbje2VANAx2u4iCo%2Bny4pCe2cMik43NgNI0HTL5oOzvgJtDx3OQub7%2FNDAqxQJmfL8TlavNDkgVDWX4D%2BkxhBSdIVn4r61uFvuQ1o%2B%2FhAUXLFGj7utRfgrC0p%2BlT2BN8l4Sgl%2F%2BkSj%2FYagvmtw6cqyiw%3D%3D

https://a.nel.cloudflare.com/report/v3?s=x%2FfpYTzbje2VANAx2u4iCo%2Bny4pCe2cMik43NgNI0HTL5oOzvgJtDx3OQub7%2FNDAqxQJmfL8TlavNDkgVDWX4D%2BkxhBSdIVn4r61uFvuQ1o%2B%2FhAUXLFGj7utRfgrC0p%2BlT2BN8l4Sgl%2F%2BkSj%2FYagvmtw6cqyiw%3D%3D