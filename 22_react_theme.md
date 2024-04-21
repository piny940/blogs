## はじめに(背景)

私のポートフォリオではライトテーマとダークテーマを切り替えできるようにしています。

https://www.piny940.com

ただ、どちらのテーマを選択しているかを localStorage に保存していたため、initial rendering の段階ではライトテーマで描画され、JS が読み込まれてからダークテーマに切り替わるという問題がありました。

JS の読み込みには 0.数秒かかるため、画面を描画する際にちらつきが発生するという問題が有りました。

そこで今回は cookie を使ってこのちらつきの問題を解消した話を紹介しようと思います。

実際のコードは Github で公開しているので参考程度にどうぞ

https://github.com/piny940/portfolio

## 環境

- Next.js 14.1.4
  - Pages Router の前提で書いていますが、App Router でも同様の方法で実装できます。

## 解決策

選択中のテーマを cookie に保存することで、サーバーサイドでのレンダリング時にも選択中のテーマを取得できるようにしました。cookie に保存されたデータは`getServerSideProps`で取得することができます。

```ts
export const getServerSideProps: GetServerSideProps<HomeProps> = async (
  ctx
) => {
  const theme = (ctx.req.cookies.theme ?? "light") as Theme
  return {
    props: {
      initialTheme: theme,
    },
  }
}
```

Bootstrap では`html`タグに`data-bs-theme`属性を追加する必要があります。ただ、`html`タグが書かれている`Document`コンポーネントでは`getServerSideProps`を使うことが出来ないため、`getInitialProps`を使って`Document`コンポーネントにテーマを渡すことにしました。

```ts
interface MyDocumentProps extends DocumentInitialProps {
  initialTheme: Theme
}
function MyDocument({ initialTheme }: MyDocumentProps) {
  return (
    <Html className="bg-body text-body" data-bs-theme={initialTheme} lang="ja">
      ...
      <body>
        <Main />
        <NextScript />
      </body>
    </Html>
  )
}
MyDocument.getInitialProps = async (
  ctx: DocumentContext
): Promise<MyDocumentProps> => {
  const initialProps = await Document.getInitialProps(ctx)
  const req = ctx.req as IncomingMessage & {
    cookies: Partial<{
      [key in string]: string
    }>
  }
  const initialTheme = (req?.cookies?.theme ?? "light") as Theme

  return { ...initialProps, initialTheme }
}
```

クライアントサイドでテーマを切り替える際には、cookie に選択中のテーマを保存するようにします。

```ts
useEffect(() => {
  if (typeof window === "undefined") return
  document.cookie = `theme=${theme};path=/`
}, [theme])
```

## 最後に

今回は cookie を使ってダークテーマ選択時に生じる画面のちらつきを解消しました。ただ、この方法は SSG では使えないため、もっといい方法はないのかなぁと感じています。
こんな方法もあるよ、という情報があれば教えていただきたいです。

## 参考文献

https://dev.to/tusharshahi/react-nextjs-dark-mode-theme-switcher-how-i-fixed-my-flicker-problem-5b54
