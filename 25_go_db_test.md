## はじめに

今回は、ポートフォリオ開発で作成したテストの中で、DBを用いたテストでした工夫を紹介します。

今回作成したコードはGithubにpushしてあるので参考程度にどうぞ

https://github.com/piny940/portfolio

ポートフォリオはこちらから確認できます。

https://www.piny940.com


## 解決策
各テストごとにトランザクションを張り、テストが終わったらロールバックすることで、データベースのデータを空にすることができます。

まず、テスト開始時にテスト用のデータベースに接続をします。

```go
func TestMain(m *testing.M) {
	dsn := fmt.Sprintf("user=%s password=%s host=%s dbname=%s sslmode=%s",
		os.Getenv("DB_USER"), os.Getenv("DB_PASSWORD"), os.Getenv("DB_HOST"),
		os.Getenv("DB_NAME"), os.Getenv("DB_SSLMODE"))
	dbClient, _ = sql.Open("postgres", dsn)
	code := m.Run()
	os.Exit(code)
}
```

テストごとにトランザクションを張り、テストが終わったらロールバックするようにします。

```go
func setup(t *testing.T) {
	t.Helper()
	t.Cleanup(teardown)
	tx := dbClient.Begin()
	db = &DB{Client: tx.Debug()}
}

func TestHoge(t *testing.T) {
  setup(t)
  // 以降テスト処理を記述
}
```

これで、テストごとにデータベースのデータを空にすることができました。

## 課題の背景

ポートフォリオのバックエンドのディレクトリ構成は次のようになっていて、`db`packageの中でデータベース周りの処理を行っていました。

<details>
<summary>ディレクトリ構成</summary>
```plaintext
.
├── Dockerfile
├── auth
│   └── jwt.go
├── config
│   ├── environments
│   │   ├── development.yaml
│   │   └── production.yaml
│   └── init.go
├── db
│   ├── action.go
│   ├── blog.go
│   ├── blog_test.go
│   ├── init.go
│   ├── init_test.go
│   ├── page.go
│   ├── project.go
│   ├── project_test.go
│   ├── tech_stack.go
│   ├── technology.go
│   └── technology_test.go
├── domain
│   ├── blog.go
│   ├── models_gen.go
│   ├── project.go
│   ├── project_test.go
│   ├── tech_stack.go
│   └── technology.go
├── go.mod
├── go.sum
├── gqlgen.yml
├── graph
│   └── generated.go
├── loader
│   ├── blog.go
│   ├── blog_test.go
│   ├── dataloader.go
│   ├── project.go
│   ├── project_test.go
│   ├── technology.go
│   └── technology_test.go
├── main.go
├── registry
│   └── registory.go
├── resolver
│   ├── auth.go
│   ├── blog.go
│   ├── common.go
│   ├── project.go
│   ├── resolver.go
│   ├── tech_stacks.go
│   └── technology.go
├── schema
│   ├── auth.gql
│   ├── blog.gql
│   ├── project.gql
│   ├── schema.gql
│   ├── tech_stacks.gql
│   └── technology.gql
├── server
│   ├── auth.go
│   ├── cors.go
│   ├── graphql.go
│   └── init.go
├── tools
│   └── tools.go
└── usecase
    ├── blog.go
    ├── project.go
    ├── project_test.go
    ├── tech_stack.go
    └── technology.go
```
</details>

`domain`や`usecase`などのpackageのテストはdbのmockを作ってユニットテストを行っていたのですが、`db`packageのテストをするためにはテスト時に直接接続するテスト用のデータベースが必要でした。

単にデータベースを作るだけであればテスト用のデータベースを用意すればいいだけなのですが、テストごとにデータベースのデータを空にしないといけないという課題がありました。

## まとめ

今回はテスト用のデータベースを構築して、テストごとにデータベースのデータを空にする方法を紹介しました。

今後はこのテスト用のデータベースを使ってE2Eテストをちゃんと書いていきたいですね
