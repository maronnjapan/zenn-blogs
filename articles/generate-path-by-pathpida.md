---
title: "pathpidaではじめるパスの自動生成"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Next.js", "pathpida", "typescript"]
published: true
---

## はじめに

Next.js において、Pages Router しかり App Router しかりディレクトリから自動で URL のパスと対応してくれるようになります。
さらにただ対応するだけでなく、pages/[id]といったように記載すれば動的な対応をしてくれます。
とても便利ですね。
一方で、コンポーネント内で遷移先の URL はどうやって記載するでしょうか？
リテラルな値を href に指定している人は結構いらっしゃるのではないでしょうか。
もちろんそれでも動くのですが、打ち間違いなどがあると望む所に遷移してくれません。
となると、コンポーネント内で使用できる URL をディレクトリから自動生成してほしいと思います。
そこで今回は[pathpida](https://github.com/aspida/pathpida)でパスを自動生成して、型安全にリンクを設定できるようにしていきます。
なお、pathpida は Nuxt.js にも対応していますが、今回は Next.js のみ扱います。

## pathpida のインストール

npm i -D pathpida でインストールします。
次に package.json に以下のような scripts を登録します。

```json
"dev:path": "pathpida --ignorepath .gitignore --enableStatic --output ./src/router"
```

オプションがなくても動きますが、今回は 3 つのオプションを設定しています。

### ignorepath オプション

自動生成する時に参照しないファイルのパスを設定します。

### enableStatic オプション

付与した場合、public ディレクトリにある静的ファイルのパスも生成します。

### output オプション

生成したい場所のディレクトリパスを設定します。
なお、生成するファイル名は[src/buildTemplate.ts](https://github.com/aspida/pathpida/blob/main/src/buildTemplate.ts#L94)を確認する限り、`$path.ts`固定で変更することは無理そうです。
以上で設定は完了です。
では実際に動かしてみます。

## 自動生成する ①：App Router の場合

ここでは App Router を使った場合の動作について確認していきます。
階層は以下のようになっています。

```json
./app
├── api
│   └── hello.ts
├── page.tsx
├── test
│   └── [test]
│       └── page.tsx
├── test2
│   └── [test2].tsx
├── [...test3]
│   └── page.tsx
└── test4
    └── index.tsx
```

この状態で先程作成した`npm run dev:path`を実行すると以下のようなファイルが生成されます。

```tsx
const buildSuffix = (url?: {
  query?: Record<string, string>;
  hash?: string;
}) => {
  const query = url?.query;
  const hash = url?.hash;
  if (!query && !hash) return "";
  const search = query ? `?${new URLSearchParams(query)}` : "";
  return `${search}${hash ? `#${hash}` : ""}`;
};
export const pagesPath = {
  _test3: (test3: string[]) => ({
    $url: (url?: { hash?: string }) => ({
      pathname: "/[...test3]" as const,
      query: { test3 },
      hash: url?.hash,
      path: `/${test3?.join("/")}${buildSuffix(url)}`,
    }),
  }),
  test: {
    _test: (test: string | number) => ({
      $url: (url?: { hash?: string }) => ({
        pathname: "/test/[test]" as const,
        query: { test },
        hash: url?.hash,
        path: `/test/${test}${buildSuffix(url)}`,
      }),
    }),
  },
  $url: (url?: { hash?: string }) => ({
    pathname: "/" as const,
    hash: url?.hash,
    path: `/${buildSuffix(url)}`,
  }),
};
export type PagesPath = typeof pagesPath;
```

一致する URL にアクセスした時に表示される page.tsx が存在する階層に合わせてオブジェクトができていますね。
なので、Pages Router で書いていたような[test2].tsx や test4/index.tsx はパスとして認識していません。
補足として buildSuffix 関数はクエリやハッシュがあればパスの後ろに付与するための処理となっています。
今回は URL にクエリが付与するケースがないので、クエリを付与することはできないようになっています。
ハッシュは$urlメソッドを呼び出す時に引数に設定すれば付与できるようになっています。
また、test3パスについては複数のパラムを動的に設定できる[…test3]という書き方になっているので、パラムは配列で指定するようになっています。
配列は``/${test3?.join('/')}${buildSuffix(url)}``から、前から順番に URL のパスとして設定されます。
実際に使うときは以下のようになります。

```tsx
import Link from "next/link";
import { pagesPath } from "@/lib/$path";
export default function Home() {
  return (
    <main>
      <Link href={pagesPath.test._test("testId").$url().path}>リンクです</Link>
      <a href={pagesPath.$url().path}>リンクです2</a>
    </main>
  );
}
```

基本には`pagesPath.ディレクトリ名.$url().path`で URL のパスを取得できます。
動的に設定できるディレクトリ名になっている[test]や[…test]の場合は、`_ディレクトリ名`のメソッドに引数を設定すると、その値に合わせてパスを生成してくれます。
動的なパラムを複数設定できる場合は、配列の先頭から順にパスを設定することは把握しなければならないですが、直感的に使用できます。
便利ですね。

## 自動生成する ①：Pages Router の場合

次は Pages Router の場合です。
階層は以下のようになっています。

```tsx
./pages
├── api
│   └── hello.ts
├── _app.tsx
├── _document.tsx
├── index.tsx
├── test
│   └── [test]
│       └── index.tsx
├── test2
│   └── [test2].tsx
├── [...test3]
│   └── index.tsx
└── test4
    └── page.tsx
```

生成してみると以下のコードが生成されます。

```tsx
export const pagesPath = {
  _test3: (test3: string[]) => ({
    $url: (url?: { hash?: string | undefined } | undefined) => ({
      pathname: "/[...test3]" as const,
      query: { test3 },
      hash: url?.hash,
    }),
  }),
  test: {
    _test: (test: string | number) => ({
      $url: (url?: { hash?: string | undefined } | undefined) => ({
        pathname: "/test/[test]" as const,
        query: { test },
        hash: url?.hash,
      }),
    }),
  },
  test2: {
    _test2: (test2: string | number) => ({
      $url: (url?: { hash?: string | undefined } | undefined) => ({
        pathname: "/test2/[test2]" as const,
        query: { test2 },
        hash: url?.hash,
      }),
    }),
  },
  test4: {
    page: {
      $url: (url?: { hash?: string | undefined } | undefined) => ({
        pathname: "/test4/page" as const,
        hash: url?.hash,
      }),
    },
  },
  $url: (url?: { hash?: string | undefined } | undefined) => ({
    pathname: "/" as const,
    hash: url?.hash,
  }),
};
export type PagesPath = typeof pagesPath;
```

大体は App Router と似た者ができますが、次の点が異なります。
①buildSuffix 関数が存在しないので、ハッシュなどを設定しても自動でそれを付与したパスが生成されないです。
②path プロパティがありません。
③api ディレクトリを除いて、pages 内にあるディレクトリはパスとして生成されます。
③ についてですが、App Router と異なり page.tsx といった指定のファイル以外が存在してもパスは生成されます。
例えば以下のようなディレクトリ構造の場合を確認します。

```tsx
./pages
├── test4
	  ├── hello2.ts
	  ├── hello.tsx
	  ├── index.tsx
	  └── page.tsx
```

パスを生成すると、以下のコードが出来上がります。

```tsx
"test4": {
    "hello": {
      $url: (url?: { hash?: string | undefined } | undefined) => ({ pathname: '/test4/hello' as const, hash: url?.hash })
    },
    "hello2": {
      $url: (url?: { hash?: string | undefined } | undefined) => ({ pathname: '/test4/hello2' as const, hash: url?.hash })
    },
    "page": {
      $url: (url?: { hash?: string | undefined } | undefined) => ({ pathname: '/test4/page' as const, hash: url?.hash })
    },
    $url: (url?: { hash?: string | undefined } | undefined) => ({ pathname: '/test4' as const, hash: url?.hash })
  },
```

どのファイルも、何なら TS ファイルについてもパスが生成されていますね。
ただ、index.tsx は実際に URL へアクセスした時表示するファイルなので、test4 プロパティの中に”index”プロパティは存在せず、そのままパスの情報にアクセスできるようになっています。
一方、他のファイルについては表示に関わらないのかパスとして生成されています。
このパス自体は実際にアクセスしても使用できないので注意です。
以上のことから、Pages Router を使う時は pages ディレクトリに表示するコンポーネントファイル以外は置かないようにした方が余計なパスを生成せずに済みそうです。
なお、生成したパスは以下のように使用します。

```tsx
import Link from "next/link";
import { pagesPath } from "@/lib/$path";
export default function Home() {
  return (
    <main>
      <Link href={pagesPath.test4.$url()}>リンク</Link>
    </main>
  );
}
```

これも便利ですが、App Router に比べると少し挙動の違いに注意する必要はあります。

### どの要素でも使用できるパスを設定する

Pages Router でパスを生成して、href に適用しましたが、実は動的なパスの場合このままだと Link コンポーネントでしか使用できません。
実際に HTML の a 要素の href に生成された動的なパスを設定しても、test/[test]といった値を設定する前のパスしか取得できません。
なので、Link コンポーネント以外で使用するにはひと手間加える必要があります。
そのためにまず任意の場所にパスを生成するファイルを作成します。
今回は$path.ts と同じ階層にファイルを作成しています。
そしたら以下のようなコードを記載します。

```tsx
import { resolveHref } from "next/dist/client/resolve-href";
import Router from "next/router";
import { pagesPath } from "./$path";
const notFoundPath = pagesPath.not_found.$url().pathname;
export const dynamicParamPath = (testId: string) =>
  resolveHref(Router, pagesPath.test._test(testId).$url(), true)[1] ??
  notFoundPath;
export const dynamicParamsPathPath = (testIds: string[]) =>
  resolveHref(Router, pagesPath._test3(testIds).$url(), true)[1] ??
  notFoundPath;
```

ここでのポイントは resolveHref 関数を使うことです。
これに Next.js の Router と生成したいパスに対応するプロパティの$urlメソッドを呼び出すことで、動的に設定した値をもとにパスを生成してくれます。
ただ、resolveHref関数は以下のように要素が一つ、もしくは二つの配列を返します。
![Untitled](/images/generate-path-by-pathpida/Untitled.png)
そして、動的な値を基に生成したパスは二つの要素の二番目に存在します。
そのため取り出す時は`resolveHref(Router, pagesPath.test._test(testId).$url(), true)[1]`のように`[1]`を指定する必要があるのですが、型定義上 undefined があり得てしまいます。
undefined を許容すると使う側での不都合が多いので、上手く動的なパスの生成ができない場合は NotFound ページに飛ばすようにしています。
ちなみに、dynamicParamsPathPath 関数のような配列で値を生成したとしても、いい感じに生成してくれます。
実際に呼び出した時の値は以下のようになります。

```tsx
console.log(dynamicParamPath("test1"));
// 出力値：/test/test1
console.log(dynamicParamsPathPath(["test2", "test3"]));
// 出力値：/test2/test3
```

以上で、Next.js の Link コンポーネント以外でも動的なパスを取得できるようになりました。
こういったタイポとかを防ぐことができるものは積極的に取り入れていきたいですね。

## 余談 $path.ts の prettier 問題

pathpida を実行した時に生成するファイルは$path.ts固定になりそうだということは先程記載しました。
ただ、この$が prettier を実行させるとき少し厄介なことになる場合があります。
具体的には[こちらのイシュー](https://github.com/prettier/prettier/issues/14987)であるように、$が特殊文字として判定されてしまうことです。
これによって、$path が「$path」の文字列ではなく、「$(特殊文字) + path」として認識されてしまい、ファイル名として認識されません。
そのため、エスケープ処理を行うなどを行い特殊文字ではなくす必要があります。
ただ、私の場合エスケープが上手くできなかったので、以下のように mv コマンドを pathpida でファイルを作成した後、ファイル名を変更することで対応しました。

```tsx
"dev:path": "pathpida --ignorepath .gitignore --enableStatic --output ./src/router && mv ./src/router/$path.ts ./src/router/path.ts"
```

mv コマンドは Linux 環境下でしか上手く動かないと思いますが、基本的に Linux 環境下で Docker を使って開発しているので、まあいいかなと思ったので mv コマンドで妥協しました。
調べる感じ mv コマンドを行わなくても、できそうなのですが上手くいっていません。
いい方法があればコメントで是非とも教えてほしいです。

## おわりに

今回は pathpida を使って動的なパス生成についてみていきました。
調べてみて驚いたのが、思ったよりパスを自動で生成するための情報が少ないことでした。
結構需要があると思うのですが、意外とないんだなと感じました。
とはいえ、pathpida 自体は特に App Router の時は使いやすいなと感じたので普段使いしようかなと思います。
ここまで読んでいただきありがとうございました。
