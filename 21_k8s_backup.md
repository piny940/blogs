## 技術選定

条件:

- 定期的に自動でバックアップがされる
- PV (PVC)もバックアップできる
- リストアが 1~2 個のコマンドで手軽にできる
- GCS に対応(S3 は無料枠が 1 年しかないため GCS を使いたい)
- メンテナンスがされていて、ドキュメントが豊富

候補:

- [Velero](https://github.com/vmware-tanzu/velero)
- longhorn で PV のみバックアップ
- [kube-backup](https://github.com/pieterlange/kube-backup)

Velero:

- Test 用のクラスタに使い回せる
- バックアップスケジューラ内包

flux を入れてるから k8s オブジェクトまで Backup ツールで復元する必要があるかは微妙

Longhorn:

- すでに使ってるリソースをそのまま使える
- 手順が複雑 volume 名が変わると PVC に対する PV 名が変わるから対応付けが難しい

kube-backup:

- メンテナンスが 5 年前からされていない

## 手順

### 1. CLI のインストール

https://velero.io/docs/v1.13/basic-install/

```
wget https://github.com/vmware-tanzu/velero/releases/download/v1.13.0/velero-v1.13.0-linux-amd64.tar.gz
tar -xvf velero-v1.13.0-linux-amd64.tar.gz
sudo mv velero-v1.13.0-linux-amd64/velero /usr/local/bin
if type _init_completion > /dev/null; then echo "bash completion has already been installed!"; else sudo apt-get update; sudo apt-get install bash-completion; fi
if type _init_completion > /dev/null; then echo "bash completion installed!"; else source /usr/share/bash-completion/bash_completion; fi
velero completion bash | sudo tee /etc/bash_completion.d/velero
```

### 2. GCP の認証情報の設定

今回はバックアップを GCP に保存するため、GCP のプラグインの README に従って認証情報を設定します。

このセクションの操作はローカル PC で行って問題ありません。最後にダウンロードしたファイルをサーバーにアップロードするのを忘れないようにしてください。

https://github.com/vmware-tanzu/velero-plugin-for-gcp?tab=readme-ov-file#Set-permissions-for-Velero

`gcloud CLI`を使って認証情報を設定するため、 ローカルマシンに`gcloud`がインストールされていない場合は[インストール](https://cloud.google.com/sdk/docs/install?hl=ja)・[初期化](https://cloud.google.com/sdk/docs/initializing?hl=ja)を行います。

まず、GCS のバケットを作成します。

```bash
BUCKET=<YOUR_BUCKET>

gsutil mb gs://$BUCKET/
```

次に、GCP のサービスアカウントを作成します。
次のコマンドで`velero`という名前のサービスアカウントを作成します。

```bash
PROJECT_ID=$(gcloud config get-value project)
GSA_NAME=velero
gcloud iam service-accounts create $GSA_NAME \
    --display-name "Velero service account"
```

`gcloud iam service-accounts list`で作成したサービスアカウントを確認できるはずです。

次に、サービスアカウントに必要な権限を付与します。

```bash
SERVICE_ACCOUNT_EMAIL=$(gcloud iam service-accounts list \
  --filter="displayName:Velero service account" \
  --format 'value(email)')
ROLE_PERMISSIONS=(
    compute.disks.get
    compute.disks.create
    compute.disks.createSnapshot
    compute.projects.get
    compute.snapshots.get
    compute.snapshots.create
    compute.snapshots.useReadOnly
    compute.snapshots.delete
    compute.zones.get
    storage.objects.create
    storage.objects.delete
    storage.objects.get
    storage.objects.list
    iam.serviceAccounts.signBlob
)

gcloud iam roles create velero.server \
    --project $PROJECT_ID \
    --title "Velero Server" \
    --permissions "$(IFS=","; echo "${ROLE_PERMISSIONS[*]}")"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:$SERVICE_ACCOUNT_EMAIL \
    --role projects/$PROJECT_ID/roles/velero.server

gsutil iam ch serviceAccount:$SERVICE_ACCOUNT_EMAIL:objectAdmin gs://${BUCKET}
```

最後にサービスアカウントの鍵を作成してダウンロードします。
次のコマンドを実行すると`credentials-velero`という名前のファイルがダウンロードされます。

```bash
gcloud iam service-accounts keys create credentials-velero \
    --iam-account $SERVICE_ACCOUNT_EMAIL
```

### 3. Velero server のインストール

Helm を使ってインストールする方法もありますが、今回は`velero install`コマンドを使ってインストールします。

`gcloud`の設定をローカル PC で行った場合は、サーバーに`credentials-velero`をアップロードしてください。

```bash
scp credentials-velero user@server:/path/to/credentials-velero
```

また、`BUCKET`の値もサーバーに設定してください。

```bash
BUCKET=<YOUR_BUCKET>
```

次のコマンドで Velero をインストールします。

```bash
velero install \
    --features=EnableCSI \
    --plugins=velero/velero-plugin-for-gcp:v1.6.0,velero/velero-plugin-for-csi:v0.7.0 \
    --provider gcp \
    --bucket $BUCKET \
    --secret-file ./credentials-velero
```

### 4. バックアップをスケジュールする

```bash
velero schedule create backup-schedule --schedule="9 3 * * *"
```

### 5. バックアップを手動で実行する

```bash
velero backup create --from-schedule backup-schedule
```

## 参考資料

https://qiita.com/ysakashita/items/3a0b2ad9ac37e2ce315a

https://qiita.com/kokohei39/items/247408c303fbbdc1c5be

https://www.wantedly.com/companies/wantedly/post_articles/135823

https://velero.io/
