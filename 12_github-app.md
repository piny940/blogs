## はじめに
前回は自己署名証明書を使ってkubernetesのingressでTLS通信ができるようにしました。

https://qiita.com/piny940/items/ca1b1ccced83708e968a

今回はkubernetesからは一旦逸れて、GithubAppを使ってコードを自動更新する仕組みを作っていこうと思います。
GithubAPIでブランチを作成してコミットするのは少し手順が複雑だったので、記事に手順を残します。

## 手順
### GithubAppを作成

[公式ドキュメント](https://docs.github.com/ja/apps/creating-github-apps/registering-a-github-app/registering-a-github-app)に従ってGithub Appを作成します。

Githubの`Settings > Developer settings > Github Apps`から作成できます。設定は適当でいいですが、App名はGithub上で一意でなくてはなりません。

GithubAppの設定画面からClientSecretとPrivate Keyを作成し、メモしておきます。

### Github Appをインストール

`Settings > Developer Settings > Github Apps`から作成したGithub Appを選択し、Install AppからGithub Appをインストールします。

### Octokitをセットアップ

まず、Octkit(https://github.com/octokit/octokit.js/?tab=readme-ov-file)をインストールします。
```bash
yarn add octkit
```
環境変数に`APP_ID`, `PRIVATE_KEY`, `CLIENT_ID`, `CLIENT_SECRET`を設定し、次のように書きます。Githubではデフォルトでダウンロードできる秘密鍵がpkcs1形式のためpkcs8形式に変換する必要があります。
```typescript
const appId = process.env.APP_ID as string
const privateKey = (process.env.PRIVATE_KEY as string).replace(
  /\\n/g,
  '\n'
)

// octokitはpkcs8形式の秘密鍵を要求する
const privateKeyPkcs8 = crypto.createPrivateKey(privateKey).export({
  type: 'pkcs8',
  format: 'pem',
}) as string

const _octokit = new Octokit({
  authStrategy: createAppAuth,
  auth: {
    appId,
    privateKey: privateKeyPkcs8,
    clientId: process.env.CLIENT_ID,
    clientSecret: process.env.CLIENT_SECRET,
  },
})
```

PullRequestの作成などにはInstallationとして認証をする必要があります。そのため、[公式ドキュメント](https://docs.github.com/ja/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation)に従ってトークンを取得します。

```typescript
const { data: installationData } = await _octokit.request(
    'GET /repos/{owner}/{repo}/installation',
    {
      owner: process.env.REPOSITORY_OWNER,
      repo: process.env.REPOSITORY_NAME,
    }
  )
  const app = new App({
    appId,
    privateKey: privateKeyPkcs8,
  })
  const octokit = await app.getInstallationOctokit(installationData.id)
```

### Commit, PRの作成
まず、分岐元のcommitを取得します。
```typescript
const { data: branchesData } = await octokit.request(
  'GET /repos/{owner}/{repo}/branches',
  {
    owner,
    repo,
  }
)
const masterBranch = branchesData.find((branch) => branch.name === process.env.BRANCH_NAME)
if (!masterBranch) throw new Error(`Branch ${process.env.BRANCH_NAME} not found`)
```

Commitを作成します。CommitはTreeを作成した後にCommitを作成します。
```typescript
const { data: tree } = await octokit.request(
  'POST /repos/{owner}/{repo}/git/trees',
  {
    owner,
    repo,
    tree: [
      {
        path: 'test.txt',
        mode: '100644',
        type: 'blob',
        content: 'test',
      },
    ],
    base_tree: masterBranch.commit.sha,
  }
)
const { data: commit } = await octokit.request(
  'POST /repos/{owner}/{repo}/git/commits',
  {
    owner,
    repo,
    message: 'test',
    tree: tree.sha,
    parents: [masterBranch.commit.sha],
  }
)
```

### ブランチの作成
ブランチは、実体はCommitへのポインタなので、先ほど作成したCommitのハッシュを指定してrefを作成します。
```typescript
const { data: ref } = await octokit.request(
  'POST /repos/{owner}/{repo}/git/refs',
  {
    owner,
    repo,
    ref: `refs/heads/branch-${Date.now()}`,
    sha: commit.sha,
  }
)
```

### Pull Requestの作成
最後にPull Requestを作成します。
```typescript
const { data: pr } = await octokit.request(
  'POST /repos/{owner}/{repo}/pulls',
  {
    owner,
    repo,
    title: 'test',
    head: ref.ref,
    base: branchName,
  }
)
```

以上でPullRequestが作成されます。

## 最後に
今回はGithub Appを用いてコードを書き換えるPullRequestを作りました。次回は今回作ったGithubAppを用いて、Qiitaから記事をAPIで読み取ってポートフォリオを自動で更新する仕組みを作っていこうと思います。

## 参考資料

https://docs.github.com/ja/apps/creating-github-apps/registering-a-github-app/registering-a-github-app

https://github.com/octokit/octokit.js/?tab=readme-ov-file

https://github.com/octokit/auth-app.js?tab=readme-ov-file#authenticate-as-github-app-json-web-token

https://github.com/gr2m/universal-github-app-jwt?tab=readme-ov-file#about-private-key-formats

https://docs.github.com/ja/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation
