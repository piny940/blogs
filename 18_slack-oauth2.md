## はじめに

前回は魔法のスプレッドシートの更新通知を流してくれるSlackAppを作成しました。

https://qiita.com/piny940/items/2f0126ad5d71775b3628

一般にSlack Appを公開するにはOAuth2.0認証をできるようにする必要があります。今回はGo言語でOAuth認証のためのサーバーを(サクッと？)立てる方法を解説していこうと思います。

完成したコードはGithub上で公開しているので参考程度にどうぞ。

https://github.com/piny940/magic-spreadsheet-notifier

## 環境

Golang 1.21.1  
labstack/echo 4.11.4  
slack-go/slack 0.12.3

今回はechoを用いてサーバーを立てましたが、Ginとかを使っても大体同じようなコードになると思うので適宜読み替えてください。

## 手順

基本的に[ドキュメント](https://api.slack.com/authentication/oauth-v2)に従い実装していきます。

### 1. 必要な環境変数を設定
SlackAppの管理画面から
- Client ID
- Client Secret

の2つを取ってきます。

取ってきた値は`.env`などに保存してGithub等にpushされないようにしておきましょう。

開発時には`.env`から環境変数を読み取れるよう[godotenv](https://github.com/joho/godotenv)を使用します。

```go
if err := godotenv.Load(".env"); err != nil {
  panic(err)
}
```

### 2. 「SlackAppを追加」ボタンを配置

aタグを配置し、リンクを次のようにします。

```typescript
`https://slack.com/oauth/v2/authorize?scope=${求める権限}&client_id=${Client IDの値}&redirect_uri=${Redirect URL}`
```
求める権限は、`channels:read`や`chat:write`などです。  
Redirect URLは、ユーザーがBotの認証を許可した場合に帰ってくるページのURLを指定します。SlackApp管理画面のRedirectURLに指定したURLの中の1つである必要があります。

ngrokなどで開発サーバーを公開しておくと開発環境でも実験が簡単にできて便利です。

### 3. コールバックのエンドポイントを作成
ユーザーがBotの認証を許可すると、RedirectURLにリダイレクトされます。このときクエリパラメータに`code`が記述されているため、これを取り出します。

```go
code := context.QueryParam("code")
```
これを用いて`oauth.v2.access`APIを叩きます。

```go
response, err := slack.GetOAuthV2ResponseContext(
  c.Request().Context(),
  new(http.Client),
  os.Getenv("SLACK_CLIENT_ID"),
  os.Getenv("SLACK_CLIENT_SECRET"),
  code,
  redirectTo,
)
```

これで上手く行くとレスポンスにはユーザーが属しているワークスペースの`AccessToken`が返ってくるので、これ以降はそのアクセストークンを用いてSlackAPIを叩くことが出来ます。

## 最後に

今回はGo言語でSlackのOAuth2.0認証をできるようにしました。OAuth認証はTwitterに対してPythonで一度実装をしたことがあったため、概念の把握はすぐにできたのですが、Go特有の「HTMLに変数を埋め込むのってどうやるんだろう」みたいなので若干苦戦しました(まぁドキュメント読めば分かる話ですが)。  
次回は、今回書くのをサボったテストを書いていこうと思います。

## 参考資料

https://api.slack.com/authentication/oauth-v2

https://echo.labstack.com

https://github.com/slack-go/slack
