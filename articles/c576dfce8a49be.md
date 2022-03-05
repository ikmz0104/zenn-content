---
title: "Next.jsのCustom Error Pageについて簡単な共有"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Next.js", "TypeScript"]
published: false
---

# はじめに
Next.jsの[エラーページの設定](https://nextjs.org/docs/advanced-features/custom-error-page)について少しはまったのでメモしておきます。

Next.jsの[公式チュートリアル](https://nextjs.org/learn/basics/create-nextjs-app)を1周した後にやると効率的です。

# 環境
WSL2
Ubuntu 20.04 on Windows(11)

# Custom Error Pageについて
[Custom Error Page](https://nextjs.org/docs/advanced-features/custom-error-page)を読んでみる。

※今回はTypeScriptを用いるので読み替え/書き換えています。

`pages/_error.tsx`を用意することでレスポンスのステータスコードに応じたページが表示されます。

ただ、カスタムページを用意してユーザに対して次のアクションを指定したいですよね。

そこで、`pages/404.tsx`と`pages/500.tsx`を用意することで`pages/_error.tsx`に入ってきたステータスコードのうち`404`と`500`だけをそれぞれのカスタムページ`pages/404.tsx`、`pages/500.tsx`にリダイレクトさせてくれます。それ以外のステータスコードは`pages/_error.tsx`に反映されます。

もちろん、`pages/_error.tsx`で`404`の場合はこの表記、`500`の場合はこの表記、それ以外のステータスコードはこの表記というような条件分岐をしてもよいですがせっかくNext.js公式さんが用意してくれたのでそのエラーハンドリング機能を使用します。

※こういうソースはあまりよくないということ。
```tsx: pages/_error.tsx
const Error: NextPage<Props> = ({ statusCode }) => {
  if(statusCode == 404){
    return(
     <Custom404 />
    )    
  } else if (statusCode == 500){
    return(
     <Custom500 />
    )    
  } else {
    return <div>{statusCode}</div>;
  }
};
```

# 動作確認
順番に設定を変更していき、無効なURLをリクエストした際にどのようなページが表示されるかを確認してみる。
```
Request URL: http://localhost:3000/aaa
Request Method: GET
Status Code: 404 Not Found
```

## 1. 初期設定では無機質なページが表示される
- 画面

![pages/に何も設定しない場合](../images/default.png)

## 2. `pages/_error.tsx`を追加するとErrorコンポーネント内のソースに応じたページが表示される
- ソース
```tsx: pages/_error.tsx
import React from 'react';
import { NextPage, NextPageContext } from 'next';
interface Props {
    statusCode: number;
}
const Error: NextPage<Props> = ({ statusCode }) => {
  return <div>{statusCode}エラーが発生しました</div>;
};

Error.getInitialProps = async ({ res, err }: NextPageContext) => {
  const statusCode = res ? res.statusCode : err ? err.statusCode ?? 500 : 404;
  return { statusCode };
};

export default Error;
```
- 画面

![pages/に_error.tsxを作成した場合](../images/error.png)

- コメント

このソースが反映されている。
```
<div>{statusCode}エラーが発生しました</div>
```


## 3. `pages/404.tsx`, `pages/500.tsx`を追加するとカスタムページが表示される
- ソース
```tsx: pages/_404.tsx
export default function Custom404() {
    return <h1>404 | Page Not Found</h1>
  }
```

```tsx: pages/_500.tsx
export default function Custom500() {
    return <h1>500 | Server-side error occurred</h1>
  }
```

- 画面

![pages/にカスタムページを作成した場合](../images/custom.png)

- コメント

`pages/_error.tsx`内のソースが反映されず、カスタムページに上書きされている。
```
<h1>404 | Page Not Found</h1>
```

# おまけ
## SSR
[Reusing the built-in error page](https://nextjs.org/docs/advanced-features/custom-error-page#reusing-the-built-in-error-page)を読んでみる。
Errorコンポーネントを使いたいページに対しては`pages/_error.tsx`ではなく`next/error`からインポートすることに注意します。
公式ではGutHub APIを用いていますが実際には他の外部APIか自作APIを使うはずなので、初期値の設定やエラーステータスのキャッチ方法などを抑えておくとよいです。
簡単なサンプルソースはこちら。

```tsx: xxx/test.tsx
import Error from 'next/error'

type Props = {
  statusCode: number;
}

export async function getServerSideProps() {
  //外部APIからデータフェッチ＆エラー系のステータスコードを取得するサンプルソース
    //const res = await fetch('https://api.github.com/repos/vercel/next.js')
    //const errorCode = res.ok ? false : res.statusCode
  //自作の内部APIからデータフェッチ＆2xx以外のエラー系のステータスコードを取得するサンプルソース
    // let statusCode = null;
    // = await Promise.all([TokenApi.getAPI($1,$2,$3).catch(error => { statusCode = error.response.status; return null;}),
  return {
    props: {
      statusCode: statusCode,
    },
  };
}
const SamplePage: React.FC<Props> = (props) => {
  if (props.statusCode) {
    return <Error statusCode={props.statusCode}></Error>
  }
  return (
    <Main />
  )
}
```

## 特定のパスをアクセス制御する
`pages/_error.tsx`のErrorコンポーネントを特定パスで呼び出すなどの設定も可能です。
かなり地道なパス指定になるのでブレークポイントを貼りながらリクエストURLがどのような値になっているかを確認して条件分岐でエラーページに強制遷移させます。
```tsx: templates/users.tsx
import { useRouter } from 'next/router'

const getTestuser: React.FC = () => {
  const router = useRouter()
  const path: string = router.pathname;
  // パスを変更したいとき
    //const path: string = router.pathname.replace('[testuserstore]', '') + testuser?.username;

  if (path == '/users/undefined') {
  //propsを渡さず直接コンポーネントを指定
    // return <Custom404 />
  //propsをErrorコンポーネントを渡す
    // return <Error statusCode={props.statusCode}></Error>
  }

```

# next-config.jsでリダイレクト設定
ページコンポーネント等を用意したくない場合は面倒な設定を避けて`next-config.js`を設定します。
ただし、あまりに強力なので消したくないパスも消えてしまう場合があります。

例えば下記の設定をしてしまうと`/users/:slug`のslugにはどんな値を入力しても`404`に飛んでしまいます。データベースに存在しているユーザも存在していないデータも`404`に飛ばすことは避けたいのでデータベース関連が付随するようなパスを指定することはおすすめできません。

```js: next.config.js
module.exports = {
  async redirects() {
    return [
      {
        source: '/users/:slug', // 受信するリクエストのパスパターン
        destination: '/404', // ルーティングしたいパス
        permanent: true, // 永続的なリダイレクトかのフラグ(一時的なリダイレクトではないならtrue)
      },
    ];
  },
};

```

# 感想
ユーザが無効なリクエストをしたときに無機質な404ページが表示されると次の手段に移行できないため、大規模ユーザを抱えるサイトではカスタムエラーページを用意することが一般的になっています。

カスタムページにはそのwebサイトのヘッダーとフッター、何が起こったのかという文言、次のアクションを指定する「TOPページに戻る」などのボタンが用意されています。

こういったカスタムページを用意するちょっとした気遣いでユーザ離れを回避できそうですね。

最後までお読みいただきありがとうございました。