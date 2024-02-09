## はじめに

Helm Releaseのインストールがなんか上手く行ってない…けどどこで何を確認したらいいか分からない！ってなったので「このコマンドでログを見よう！」ってのをメモっておきます。

fluxを使っていること前提で書いています。

## 試すコマンド

- `flux logs`  
fluxにマニフェストが正しく認識されているか確認します。
- `flux logs -n ...`
flux上の問題によりエラーが生じていないか確認します。
- `kubectl get helmreleases`
HelmReleaseが正常にインストールできているか確認します。
- `kubectl describe helmrelease ... -n ...`
上のコマンドで正常にインストールできていなかった場合はその原因を確認します。
- `kubectl get pods -n ...`
Podが正常に作られているか確認します。
- `kubectl describe pod -n ...` & `kubectl logs ... -n ...`
Podのログを確認します。
