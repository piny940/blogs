## はじめに

本記事では Kubernetes クラスタのバックアップを取る方法を紹介します。

普段運用している Kubernetes クラスタでは fluxCD を使用しているため、単にオブジェクトを復元するだけであれば fluxCD の機能を使えばよかったのですが、Volume のバックアップも取りたいという要望があったため、バックアップツールを導入することにしました。

## 技術選定

使用するツールの条件として以下のようなものを考えました。

- 定期的に自動でバックアップがされる
- PV (PVC)もバックアップできる
- リストアが 1~2 個のコマンドで手軽にできる
- メンテナンスがされていて、ドキュメントが豊富

これを踏まえて次の 3 つを候補として考えました。

- [Velero](https://github.com/vmware-tanzu/velero)
- longhorn によるバックアップ
- [kube-backup](https://github.com/pieterlange/kube-backup)

| ツール      | メリット                                                                                                          | デメリット                                                                                                                         |
| ----------- | ----------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Velero      | Test 用のクラスタに使い回せる<br>バックアップスケジューラ内包                                                     | flux を入れてるのに k8s オブジェクトまで Backup ツールで復元する必要ことになってちょっと微妙(それは他のツールでも同じ？)           |
| longhorn    | すでにストレージに使っている longhorn をそのまま使える<br/>友人が longhorn でバックアップを取っていて参考にできる | 別のストレージサービスを使うことになった場合別途バックアップをする必要がある(?)<br/>バックアップとストレージは疎結合にしておきたい |
| kube-backup |                                                                                                                   | メンテナンスが 5 年前からされていない                                                                                              |

今後 longhorn 以外のストレージサービスを使う可能性もあるため、今回は Velero を選択しました。

## 手順

### 1. CLI のインストール

参考: https://velero.io/docs/v1.13/basic-install/

```bash
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

参考: https://github.com/vmware-tanzu/velero-plugin-for-gcp?tab=readme-ov-file#Set-permissions-for-Velero

`gcloud CLI`を使って認証情報を設定するため、 ローカルマシンに`gcloud`がインストールされていない場合は[インストール](https://cloud.google.com/sdk/docs/install?hl=ja)・[初期化](https://cloud.google.com/sdk/docs/initializing?hl=ja)を行います。

GCS のバケットを作成:

```bash
BUCKET=<YOUR_BUCKET>

gsutil mb gs://$BUCKET/
```

GCP のサービスアカウントを作成:  
次のコマンドで`velero`という名前のサービスアカウントを作成します。

```bash
PROJECT_ID=$(gcloud config get-value project)
GSA_NAME=velero
gcloud iam service-accounts create $GSA_NAME \
    --display-name "Velero service account"
```

`gcloud iam service-accounts list`で作成したサービスアカウントを確認できるはずです。

サービスアカウントに必要な権限を付与:

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

サービスアカウントの鍵を作成してダウンロード:  
`credentials-velero`という名前のファイルがダウンロードされます。

```bash
gcloud iam service-accounts keys create credentials-velero \
    --iam-account $SERVICE_ACCOUNT_EMAIL
```

### 3. Velero server のインストール

`gcloud`の設定をローカル PC で行った場合は、サーバーに`credentials-velero`をアップロードします。

```bash
scp credentials-velero user@server:/path/to/credentials-velero.json
```

インストール方法には helm を使う方法と`velero install`を使う方法があります。今回は Helm を使う方法をとりました。

`velero`という名前の namespace を作成:

```bash
kubectl create namespace velero
```

サーバーにアップロードした鍵をもとに secret を作成:

```bash
kubectl create secret generic google-credentials -n velero --from-file=gcp=./credentials-velero.json
```

Helm 用の`values.yaml`を作成:

ポイントは次の 5 点です。

1. `initContainers`に`velero-plugin-for-csi`と`velero-plugin-for-gcp`を追加
1. `configuration.features`に`EnableCSI`を追加
1. `nodeAgent`を設定・`deployNodeAgent`を`true`にする
1. `schedules`で`snapshotVolume`を`true`にする
1. `defaultSnapshotMoveData: true`を指定する

これによってボリュームのスナップショットもバックアップされるようになります。

```yaml
image:
  repository: velero/velero
  tag: v1.13.0
  pullPolicy: IfNotPresent
resources:
  requests:
    cpu: 500m
    memory: 128Mi
  limits:
    cpu: 1000m
    memory: 512Mi
initContainers:
  - name: velero-plugin-for-csi
    image: velero/velero-plugin-for-csi:v0.7.1
    imagePullPolicy: IfNotPresent
    volumeMounts:
      - mountPath: /target
        name: plugins
  - name: velero-plugin-for-gcp
    image: velero/velero-plugin-for-gcp:v1.9.1
    volumeMounts:
      - mountPath: /target
        name: plugins
configuration:
  backupStorageLocation:
    - name: default
      provider: velero.io/gcp
      bucket: velero-piny940
      credential:
        name: google-credentials
        key: gcp
  volumeSnapshotLocation:
    - name: default
      provider: velero.io/gcp
      credential:
        name: google-credentials
        key: gcp
  features: EnableCSI
  defaultSnapshotMoveData: true
deployNodeAgent: true
nodeAgent:
  podVolumePath: /var/lib/kubelet/pods
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
schedules:
  backup-daily:
    disabled: false
    schedule: "48 3 * * *"
    template:
      labels:
        app: velero-backup-daily
      snapshotVolumes: true
```

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm install velero vmware-tanzu/velero --namespace velero --values velero/values.yaml
```

### 4. バックアップを手動で実行する

velero のインストールが完了したら、正常にバックアップを作成できるかチェックします。

```bash
velero backup create --from-schedule backup-schedule
```

### 5. バックアップのリストア

バックアップをリストアするには、`velero restore`コマンドを使います。

```bash
velero restore create --from-backup <backup-name>
```

上手く行けば、バックアップがリストアされます。

## 最後に

今回は Velero を使ってクラスタのバックアップを取る方法を紹介しました。
これでサーバーが壊れても安心です！
次回は kubernetes のログ周りをいい感じに整えたいと思います。

## 付録

### エラー対応

バックアップを作成しようとしたら、`VolumeSnapshotClass`が見つからないというエラーが発生しました。

```
k logs velero-7ff79d67b6-5bn6d -n velero | grep error
Defaulted container "velero" out of: velero, velero-velero-plugin-for-gcp (init), velero-velero-plugin-for-csi (init)
time="2024-04-22T02:18:36Z" level=error msg="Current BackupStorageLocations available/unavailable/unknown: 0/0/1)" controller=backup-storage-location logSource="pkg/controller/backup_storage_location_controller.go:180"
time="2024-04-22T02:25:14Z" level=info msg="1 errors encountered backup up item" backup=velero/backup-20240422112304 logSource="pkg/backup/backup.go:457" name=postgres-cluster-0
time="2024-04-22T02:25:14Z" level=error msg="Error backing up item" backup=velero/backup-20240422112304 error="error executing custom action (groupResource=persistentvolumeclaims, namespace=database, name=pgdata-postgres-cluster-0): rpc error: code = Unknown desc = failed to get volumesnapshotclass for storageclass longhorn: error listing volumesnapshot classes: the server could not find the requested resource (get volumesnapshotclasses.snapshot.storage.k8s.io)" logSource="pkg/backup/backup.go:461" name=postgres-cluster-0
time="2024-04-22T02:25:14Z" level=info msg="1 errors encountered backup up item" backup=velero/backup-20240422112304 logSource="pkg/backup/backup.go:457" name=data-postgresql-0
time="2024-04-22T02:25:14Z" level=error msg="Error backing up item" backup=velero/backup-20240422112304 error="error executing custom action (groupResource=persistentvolumeclaims, namespace=database, name=data-postgresql-0): rpc error: code = Unknown desc = failed to get volumesnapshotclass for storageclass longhorn: error listing volumesnapshot classes: the server could not find the requested resource (get volumesnapshotclasses.snapshot.storage.k8s.io)" error.file="/go/src/github.com/vmware-tanzu/velero/pkg/backup/item_backupper.go:380" error.function="github.com/vmware-tanzu/velero/pkg/backup.(*itemBackupper).executeActions" logSource="pkg/backup/backup.go:461" name=data-postgresql-0
time="2024-04-22T02:25:16Z" level=error msg="no matches for kind \"VolumeSnapshot\" in version \"snapshot.storage.k8s.io/v1\"" backup=velero/backup-20240422112304 logSource="pkg/backup/snapshots.go:56"
time="2024-04-22T02:25:17Z" level=error msg="no matches for kind \"VolumeSnapshotContent\" in version \"snapshot.storage.k8s.io/v1\"" backup=velero/backup-20240422112304 logSource="pkg/backup/snapshots.go:61"
time="2024-04-22T02:25:19Z" level=error msg="cannot list VolumeSnapshotClass no matches for kind \"VolumeSnapshotClass\" in version \"snapshot.storage.k8s.io/v1\"" backup=velero/backup-20240422112304 error="no matches for kind \"VolumeSnapshotClass\" in version \"snapshot.storage.k8s.io/v1\"" logSource="internal/volume/volumes_information.go:468"
```

[external-snapshotter](https://github.com/kubernetes-csi/external-snapshotter/tree/master?tab=readme-ov-file)から CRD やら xx-controller やらをインストールする必要があるようです。

```bash
$ git clone git@github.com:kubernetes-csi/external-snapshotter.git
$ cd external-snapshotter
$ kubectl kustomize client/config/crd | kubectl create -f -
$ kubectl kustomize deploy/kubernetes/csi-snapshotter | kubectl create -f -
```

これを実行すると、無事「CRD がないよ」系のエラーが解消されました。

次に以下のようなエラーが発生しました。

```
Errors:
  Velero:    name: /postgres-cluster-0 message: /Error backing up item error: /error executing custom action (groupResource=persistentvolumeclaims, namespace=database, name=pgdata-postgres-cluster-0): rpc error: code = Unknown desc = failed to get volumesnapshotclass for storageclass longhorn: error getting volumesnapshotclass: failed to get volumesnapshotclass for provisioner driver.longhorn.io, ensure that the desired volumesnapshot class has the velero.io/csi-volumesnapshot-class label
             name: /data-postgresql-0 message: /Error backing up item error: /error executing custom action (groupResource=persistentvolumeclaims, namespace=database, name=data-postgresql-0): rpc error: code = Unknown desc = failed to get volumesnapshotclass for storageclass longhorn: error getting volumesnapshotclass: failed to get volumesnapshotclass for provisioner driver.longhorn.io, ensure that the desired volumesnapshot class has the velero.io/csi-volumesnapshot-class label
```

`VolumeSnapshotClass`がないよというエラーだったので、[longhorn](https://longhorn.io/docs/archives/1.4.1/snapshots-and-backups/csi-snapshot-support/csi-volume-snapshot-associated-with-longhorn-snapshot/)と[velero](https://velero.io/docs/main/csi/#implementation-choices)のドキュメントを参考に`VolumeSnapshotClass`を作成しました。

```yaml
kind: VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
metadata:
  name: longhorn-snapshot-vsc
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: driver.longhorn.io
deletionPolicy: Delete
parameters:
  type: snap
```

すると無事バックアップが作成されました。

```
$ velero backup describe backup-20240422225629 --details
Name:         backup-20240422225629
Namespace:    velero
Labels:       velero.io/storage-location=default
Annotations:  velero.io/resource-timeout=10m0s
              velero.io/source-cluster-k8s-gitversion=v1.28.4
              velero.io/source-cluster-k8s-major-version=1
              velero.io/source-cluster-k8s-minor-version=28

Phase:  Completed


Namespaces:
  Included:  database
  Excluded:  <none>

Resources:
  Included:        *
  Excluded:        <none>
  Cluster-scoped:  auto

Label selector:  <none>

Or label selector:  <none>

Storage Location:  default

Velero-Native Snapshot PVs:  auto
Snapshot Move Data:          false
Data Mover:                  velero

TTL:  720h0m0s

CSISnapshotTimeout:    10m0s
ItemOperationTimeout:  4h0m0s

Hooks:  <none>

Backup Format Version:  1.1.0

Started:    2024-04-22 13:57:11 +0000 UTC
Completed:  2024-04-22 13:57:40 +0000 UTC

Expiration:  2024-05-22 13:57:11 +0000 UTC

Total items to be backed up:  65
Items backed up:              65

Backup Item Operations:
  Operation for volumesnapshots.snapshot.storage.k8s.io database/velero-pgdata-postgres-cluster-0-bzw78:
    Backup Item Action Plugin:  velero.io/csi-volumesnapshot-backupper
    Operation ID:               database/velero-pgdata-postgres-cluster-0-bzw78/2024-04-22T13:57:21Z
    Items to Update:
              volumesnapshots.snapshot.storage.k8s.io database/velero-pgdata-postgres-cluster-0-bzw78
    Phase:    Completed
    Created:  2024-04-22 13:57:21 +0000 UTC
    Started:  2024-04-22 13:57:21 +0000 UTC
  Operation for volumesnapshotcontents.snapshot.storage.k8s.io /snapcontent-c8934130-f792-46ea-b03b-91028827145f:
    Backup Item Action Plugin:  velero.io/csi-volumesnapshotcontent-backupper
    Operation ID:               snapcontent-c8934130-f792-46ea-b03b-91028827145f/2024-04-22T13:57:21Z
    Items to Update:
              volumesnapshotcontents.snapshot.storage.k8s.io /snapcontent-c8934130-f792-46ea-b03b-91028827145f
    Phase:    Completed
    Created:  2024-04-22 13:57:21 +0000 UTC
    Started:  2024-04-22 13:57:21 +0000 UTC
  Operation for volumesnapshots.snapshot.storage.k8s.io database/velero-data-postgresql-0-8p4rg:
    Backup Item Action Plugin:  velero.io/csi-volumesnapshot-backupper
    Operation ID:               database/velero-data-postgresql-0-8p4rg/2024-04-22T13:57:31Z
    Items to Update:
              volumesnapshots.snapshot.storage.k8s.io database/velero-data-postgresql-0-8p4rg
    Phase:    Completed
    Created:  2024-04-22 13:57:31 +0000 UTC
    Started:  2024-04-22 13:57:31 +0000 UTC
  Operation for volumesnapshotcontents.snapshot.storage.k8s.io /snapcontent-3a9daaf4-180c-478a-98fb-170e0d307f6e:
    Backup Item Action Plugin:  velero.io/csi-volumesnapshotcontent-backupper
    Operation ID:               snapcontent-3a9daaf4-180c-478a-98fb-170e0d307f6e/2024-04-22T13:57:31Z
    Items to Update:
              volumesnapshotcontents.snapshot.storage.k8s.io /snapcontent-3a9daaf4-180c-478a-98fb-170e0d307f6e
    Phase:    Completed
    Created:  2024-04-22 13:57:31 +0000 UTC
    Started:  2024-04-22 13:57:31 +0000 UTC
Resource List:
  acid.zalan.do/v1/OperatorConfiguration:
    - database/postgres-operator
  acid.zalan.do/v1/postgresql:
    - database/postgres-cluster
  apiextensions.k8s.io/v1/CustomResourceDefinition:
    - externalsecrets.external-secrets.io
    - helmcharts.source.toolkit.fluxcd.io
    - helmreleases.helm.toolkit.fluxcd.io
    - helmrepositories.source.toolkit.fluxcd.io
    - operatorconfigurations.acid.zalan.do
    - postgresqls.acid.zalan.do
  apps/v1/ControllerRevision:
    - database/cluster-db-6b79d5f59b
    - database/cluster-db-ccc5c994
    - database/postgres-cluster-5cb485c687
    - database/postgres-cluster-6968f7b6f6
    - database/postgres-cluster-745869b67f
    - database/postgres-cluster-766df988cb
    - database/postgres-cluster-79d58c54bd
    - database/postgres-cluster-7ff9f7fd5b
    - database/postgres-cluster-bdd5ddf64
  apps/v1/Deployment:
    - database/postgres-operator
  apps/v1/ReplicaSet:
    - database/postgres-operator-7f6b6fb948
    - database/postgres-operator-8694bf465b
  apps/v1/StatefulSet:
    - database/postgres-cluster
  discovery.k8s.io/v1/EndpointSlice:
    - database/cluster-db-repl-68hjl
    - database/postgres-cluster-repl-85mrq
    - database/postgres-cluster-vpsvp
    - database/postgres-operator-fcmgp
  external-secrets.io/v1beta1/ExternalSecret:
    - database/postgres-secret
  helm.toolkit.fluxcd.io/v2beta2/HelmRelease:
    - database/postgres-operator
  policy/v1/PodDisruptionBudget:
    - database/postgres-postgres-cluster-pdb
  rbac.authorization.k8s.io/v1/ClusterRole:
    - postgres-operator
  rbac.authorization.k8s.io/v1/ClusterRoleBinding:
    - postgres-operator
  rbac.authorization.k8s.io/v1/RoleBinding:
    - database/postgres-pod
  snapshot.storage.k8s.io/v1/VolumeSnapshot:
    - database/velero-data-postgresql-0-8p4rg
    - database/velero-pgdata-postgres-cluster-0-bzw78
  snapshot.storage.k8s.io/v1/VolumeSnapshotClass:
    - longhorn-snapshot-vsc
  snapshot.storage.k8s.io/v1/VolumeSnapshotContent:
    - snapcontent-3a9daaf4-180c-478a-98fb-170e0d307f6e
    - snapcontent-c8934130-f792-46ea-b03b-91028827145f
  source.toolkit.fluxcd.io/v1beta2/HelmChart:
    - database/database-postgres-operator
  source.toolkit.fluxcd.io/v1beta2/HelmRepository:
    - database/postgres-operator
  v1/ConfigMap:
    - database/kube-root-ca.crt
  v1/Endpoints:
    - database/postgres-cluster
    - database/postgres-cluster-config
    - database/postgres-cluster-repl
    - database/postgres-operator
  v1/Event:
    - database/database-postgres-operator.17bcfc3029c627b4
    - database/postgres-operator.17c0eda01d92b131
    - database/postgres-secret.17c76139cfb06c55
  v1/Namespace:
    - database
  v1/PersistentVolume:
    - pvc-139d2fbd-6d0a-465c-bcbe-5acf54a0a246
    - pvc-95bea64e-b0c6-4628-8eef-66b1e68aa2fe
  v1/PersistentVolumeClaim:
    - database/data-postgresql-0
    - database/pgdata-postgres-cluster-0
  v1/Pod:
    - database/postgres-cluster-0
    - database/postgres-operator-7f6b6fb948-l65qc
  v1/Secret:
    - database/postgres-secret
    - database/postgres.postgres-cluster.credentials.postgresql.acid.zalan.do
    - database/sh.helm.release.v1.postgres-operator.v1
    - database/sh.helm.release.v1.postgres-operator.v2
    - database/standby.postgres-cluster.credentials.postgresql.acid.zalan.do
  v1/Service:
    - database/postgres-cluster
    - database/postgres-cluster-config
    - database/postgres-cluster-repl
    - database/postgres-operator
  v1/ServiceAccount:
    - database/default
    - database/postgres-operator
    - database/postgres-pod

Backup Volumes:
  Velero-Native Snapshots: <none included>

  CSI Snapshots:
    database/data-postgresql-0:
      Snapshot:
        Operation ID: database/velero-data-postgresql-0-8p4rg/2024-04-22T13:57:31Z
        Snapshot Content Name: snapcontent-3a9daaf4-180c-478a-98fb-170e0d307f6e
        Storage Snapshot ID: snap://pvc-139d2fbd-6d0a-465c-bcbe-5acf54a0a246/snapshot-3a9daaf4-180c-478a-98fb-170e0d307f6e
        Snapshot Size (bytes): 10737418240
        CSI Driver: driver.longhorn.io
    database/pgdata-postgres-cluster-0:
      Snapshot:
        Operation ID: database/velero-pgdata-postgres-cluster-0-bzw78/2024-04-22T13:57:21Z
        Snapshot Content Name: snapcontent-c8934130-f792-46ea-b03b-91028827145f
        Storage Snapshot ID: snap://pvc-95bea64e-b0c6-4628-8eef-66b1e68aa2fe/snapshot-c8934130-f792-46ea-b03b-91028827145f
        Snapshot Size (bytes): 10737418240
        CSI Driver: driver.longhorn.io

  Pod Volume Backups: <none included>

HooksAttempted:  0
HooksFailed:     0
```

## 参考資料

https://qiita.com/ysakashita/items/3a0b2ad9ac37e2ce315a

https://qiita.com/kokohei39/items/247408c303fbbdc1c5be

https://www.wantedly.com/companies/wantedly/post_articles/135823

https://velero.io/
