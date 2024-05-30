## はじめに

longhorn はデフォルトで 25%のディスクに対して Volume を作成しないようになっています。

![Longhorn UI](https://longhorn.io/img/screenshots/getting-started/longhorn-ui.png)

このため、ディスクの容量が 100GB ある場合、Volume は 75GB までしか作成できません。

これではストレージが無駄になってしまいます。そこでこの記事では、longhorn の設定を変更して、予約されたディスク容量を変更する方法を紹介します。

## 方法

longhorn の設定を変更するために、Helm の `values.yaml` ファイルを編集します。

```yaml
defaultSettings:
  storageReservedPercentageForDefaultDisk: 5
```

この設定を変更することで、デフォルトのディスクに対する予約容量を変更できます。デフォルトでこの数字は 25(%)になっています。今回は 5(%)に変更します。

これでディスク容量の 5%が予約されるようになります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3330232/3bc3b635-c8f5-30d0-30e5-b4794245d557.png)

## 参考資料

https://github.com/longhorn/charts/blob/v1.6.x/charts/longhorn/values.yaml#L205

https://stackoverflow.com/questions/70741359/longhorn-using-more-than-50-of-storage-as-reserved-space
