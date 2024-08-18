## はじめに

前回は Alertmanager を使って、Kubernetes で異常が生じた際に、Slack にアラートが流れるようにしました。

https://qiita.com/piny940/items/1e44d8caf936216ca93e

しかし、Alertmanager を Kubernetes クラスタ上に立てているため、クラスタごとダウンした際にはそれをアラートで気づくことができないという課題があります。

そこで今回は、Alertmanager のサービスの起動状況を監視する lambda を使用して、Alertmanager がダウンした時に Slack にアラートが送信されるようにしていこうと思います。

完成したコードはこちらから確認できます。

https://github.com/piny940/infra/tree/main/service-monitor

## 実装

コード自体は、Alertmanager のヘルスチェック用のエンドポイントにリクエストを送って、200 が返ってくるかを確認するだけのものです。

3 回リクエストをして、3 回とも失敗したらアラートを Slack に送るようにしています。

```py
import requests
import os
from slack_sdk import WebClient


def notify_slack():
  slack_token = os.getenv('SLACK_API_TOKEN')
  if slack_token == None:
    print('SLACK_API_TOKEN environment variable is not set')
    exit(1)
  client = WebClient(token=slack_token)

  channel = os.getenv('SLACK_CHANNEL')
  if channel == None:
    print('SLACK_CHANNEL environment variable is not set')
    exit(1)
  client.chat_postMessage(channel=channel, text='Alert Manager is down! :fire:')


def check(path):
  res = requests.get(path)
  if res.status_code == 200:
    return True
  else:
    return False


def handler(*_):
  path = os.getenv('HEALTH_CHECK_URL')
  if path == None:
    print('HEALTH_CHECK_URL environment variable is not set')
    exit(1)

  check_count = int(os.getenv('CHECK_COUNT') or '3')
  ok = False

  print('Checking health of service...')
  for _ in range(check_count):
    ok = check(path)
    if ok:
      break

  if ok:
    print("Service is up and running")
  else:
    notify_slack()
    print('Service is down. Alert sent to slack')
```

これを lambda で定期実行することで、Alertmanager のサービスの起動状況を監視することができます。

## まとめ

今回は、Alertmanager のサービスの起動状況を監視する lambda を作成しました。  
これで k8s クラスタ自体がダウンした際にも Slack にアラートが送信されるようになりました。

ここまでで、サービスを監視する仕組みを粗方整えることができました。  
そのため、次回はいよいよ[セトリ検索サイト](https://song-list.piny940.com)を Kubernetes に移行していきたいと思います。
