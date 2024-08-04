---
title: "テストを考えるとts-restを使うのを断念したけど、対応策みたいなのもあってどうしようか悩ましいという話→解決策を教えていただきました"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ts-rest", "jest"]
published: true
---

## はじめに

最近個人開発で[ts-rest](https://ts-rest.com/)を使っていました。
疎通確認もでき、少しの API も作りました。
さあどんどんアプリを作ろうと思ったのですが、結局断念しました。
今回はその理由について見ていきます。

## この記事のここだけ!!!

NestJS でテストコードを書く時、ts-rest を使った場合戻り値の型定義ができません。
公式は supertest で書くことを想定しており、型補完は考慮していないです。
ただ、アサーションとかで妥協案的な対応はできそうです。

## ts-rest とは

ts-rest は以下の特徴を持つ API 定義実装ができるライブラリとなっています。

- 最初から最後まで型の安全性がある
- RPC のようにフロント側で API を呼び出せる
- 軽い(ドキュメントでは 1kb と謳っています)
- コード生成が行われない
- Zod によるリクエストなどの型チェックを行える
- OpenAPI にも対応している

売りとしてはとにかく簡単に型を付与しつつ、API の定義と呼び出しができることにあります。
説明は以上になります。
とても簡素ですが、この簡素さが使いやすさに繋がっています。
少ししか触っていないですが、実装のしやすさと何をしているかの把握はかなりしやすいなと感じています。
それを体験して頂くために、簡単な実装をしていきます。

## 簡単な実装

### 実装に入る前の注意点

今回使用しているのはバックエンドが NestJS でフロントエンドが React です。
そして、自分が使ったサンプルリポジトリではモノレポ構成になっています。
ですが、この辺については解説しません。
準備についてはほぼ「[npm workspace を利用して NestJS + Create React App をモノレポ化しよう](https://note.com/shift_tech/n/nbecb007ac2ee)」の通りになっているからです。
ただ、React は Vite で作成していますので、そこは違います。
上記記事をもとに、以下のようなディレクトリ構成を構築しています。

```
.
├── backend(NestJS) 
├── frontend(React)  
└── packages
    └── ts-router(ts-rest)
```

では始めます。

### インストール

まずはバックエンドに`@ts-rest/nest`を、フロントエンドに`@ts-rest/react-query`、packages の ts-router に`@ts-rest/core`と`zod`をインストールします。
インストールの仕方は各プロジェクトのやり方に沿えば良いですが、一応私は以下の npm コマンドでインストールしています。

```
# バックエンド
npm i @ts-rest/nest -w バックエンドのpackage.json内のnameプロパティの値
# フロントエンド
npm i @ts-rest/nest -w フロントエンドのpackage.json内のnameプロパティの値
# ts-router
npm i @ts-rest/nest -w ts-routerのpackage.json内のnameプロパティの値
```

### API 定義の準備

ts-router ディレクトリ内に任意のファイルを作成し ts-rest についてのコードを記載します。
以下はサンプルです。

```tsx
//index.ts
export * from "./contract";
//contract.ts
import { initContract } from "@ts-rest/core";
import { z } from "zod";
const c = initContract();
export const tsRestRoute = c.router({
  hello: {
    method: "GET",
    path: "/api",
    responses: {
      200: z.object({
        name: z.string(),
      }),
    },
  },
});
```

基本的には以下の項目を記載します。
①HTTP メソッド
②API のパス
③ 戻り値
④ リクエストボディ(GET メソッド以外のみ)
これだけで定義は完了です。
サンプルでは reponses に zod を使っていますが、リクエストボディに zod を使えます。
これによって型が一致しないときはエラーを発生させることができ、型安全に通信を行えます。
後は先程参考にした記事の[こちら](https://note.com/shift_tech/n/nbecb007ac2ee#:~:text=%E3%83%90%E3%83%83%E3%82%AF%E3%82%A8%E3%83%B3%E3%83%89%E3%81%A8%E3%83%95%E3%83%AD%E3%83%B3%E3%83%88%E3%82%A8%E3%83%B3%E3%83%89%E3%81%8B%E3%82%89%E5%85%B1%E9%80%9A%E3%83%A2%E3%82%B8%E3%83%A5%E3%83%BC%E3%83%AB%E3%82%92%E5%88%A9%E7%94%A8%E3%81%99%E3%82%8B%E3%80%82)を参考にして、フロントエンドとバックエンドにインストールするようにします。

### バックエンドの実装

次にバックエンドの実装をします。
基本的に[ドキュメント](https://ts-rest.com/docs/nest/)の通りです。
ただ、ドキュメントではしれっと contract.ts の実装が記載されていません。
なので、まず以下のように contract.ts をバックエンドの src 配下に作成します。

```tsx
import { tsRestRoute } from "@monorepo-oidc-app/ts-router/dist";
import { nestControllerContract } from "@ts-rest/nest";
export const c = nestControllerContract(tsRestRoute);
```

実はこれだけで準備完了です。
後はドキュメントを参考にしつつ、以下のようなメソッドをコントローラーファイルに実装します。

```tsx
import { Controller } from "@nestjs/common";
import { TsRestHandler, tsRestHandler } from "@ts-rest/nest";
import { c } from "./contract";
@Controller()
export class AppController {
  @TsRestHandler(c.hello)
  async getHello() {
    return tsRestHandler(c.hello, async () => {
      return {
        status: 200,
        body: { name: "Hello" },
      };
    });
  }
}
```

余談ですが、ts-router で定義したリクエストボディなどリクエストから受け取る値を使用するには、tsrestHandler 関数の第二引数の引数に記載します。

```tsx
tsRestHandler(c.hello, async ({ body }) => {
  //...略
});
```

また、Response オブジェクトや Request オブジェクトを使用したい場合は、NestJS で定義したメソッドの引数に記載します。([ドキュメント](https://ts-rest.com/docs/nest/#using-nest-decorators)参照)

### フロントエンドの実装

まずは任意のファイルを作成し、以下のコードを記載します。

```tsx
import { tsRestRoute } from "@monorepo-oidc-app/ts-router";
import { initQueryClient } from "@ts-rest/react-query";
export const tsRestClient = initQueryClient(tsRestRoute, {
  baseUrl: window.location.origin + "/api",
  baseHeaders: {},
});
```

次に、vite.config.ts を以下のように変更します。

```tsx
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
// https://vitejs.dev/config/
export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      "/api": {
        target: "http://localhost:3000",
      },
    },
  },
});
```

補足として、ts-rest に関わるのは`initQueryClient(tsRestRoute`の部分だけです。
それ以外は Cors エラーを解消するために、Vite の設定を行っているだけで ts-rest と直接は関係ありません。
後は React コンポーネントで以下のように呼び出します。

```tsx
import { tsRestClient } from "./ts-rest-client";
function App() {
  const { data } = tsRestClient.hello.useQuery(["hello"]);
  return <div>{data?.body.name}</div>;
}
export default App;
```

通常の React Query ぽく使えますね。
これで React と NestJS を起動すると以下の画面が表示されます。
![2024-02-22_00h35_59.png](/images/ts-rest-and-jest/2024-02-22_00h35_59.png)
ちなみに API から取得したデータを使えるようにするため、TanStack Query といった React Query 系を使用可能にする必要があります。
今回は[TanStack Query](https://tanstack.com/)を用いました。
準備の仕方は[クイックスタート](https://tanstack.com/query/latest/docs/framework/react/quick-start)を参考にしてください。
ちなみに私は以下の設定だけで行けました。

```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App.tsx";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
const queryClient = new QueryClient();
ReactDOM.createRoot(document.getElementById("root")!).render(
  <QueryClientProvider client={queryClient}>
    <React.StrictMode>
      <App />
    </React.StrictMode>
  </QueryClientProvider>
);
```

以上でクイックスタート的なことが完了しました。
結構簡単で、API 定義を一つのファイルにまとめることができるのは管理がしやすいようにも感じますね。
大規模になっていくとまた色々考慮することはあるかもしれませんが、現状個人開発でそこまで規模が大きくなることは考えにくいです。
となると、ts-rest はとても良い選択肢に思えますが、ts-rest で開発を続けるには看過できない問題があったため、断念しました。
それでは、なぜ断念したかを確認します。

## ts-rest を断念した理由

ここからは本題の ts-rest の使用を断念した理由をみていきます。
先に結論を言うと、テストコードを書くのが困難だと感じたためです。
まずは先程作成した getHello メソッドに対するテストコードを書いてみます。

```tsx
import { Test, TestingModule } from "@nestjs/testing";
import { AppController } from "./app.controller";
import { PrismaService } from "./prisma/prisma.service";
import { c } from "./contract";
import * as z from "zod";
describe("AppController", () => {
  let appController: AppController;
  beforeEach(async () => {
    const app: TestingModule = await Test.createTestingModule({
      controllers: [AppController],
      providers: [PrismaService],
    }).compile();
    appController = app.get<AppController>(AppController);
  });
  it("値が取得できること", async () => {
    const result = await appController.getHello();
    const data = await result({ headers: {} });
    expect(data.body).toEqual({ name: "Hello" });
  });
});
```

これでテストは通ります。
なら別にいいのではと思いますが、以下の画像をご覧ください。
![Untitled](/images/ts-rest-and-jest/Untitled.png)
body が unknown になっています!!!
これは戻り値の型が記載されている[libs/ts-rest/core/src/lib/infer-types.ts](https://github.com/ts-rest/ts-rest/blob/main/libs/ts-rest/core/src/lib/infer-types.ts#L85).を確認すると、以下のコードの最後に分類されているのが原因ぽそうです。

```tsx
| (Or<
      Extends<TStrictStatusCodes, 'force'>,
      And<
        Extends<T, AppRouteStrictStatusCodes>,
        Not<Extends<TStrictStatusCodes, 'ignore'>>
      >
    > extends true
      ? never
      : Exclude<TStatus, keyof T['responses']> extends never
      ? never
      : {
          status: Exclude<TStatus, keyof T['responses']>;
          body: unknown;
        } & (TClientOrServer extends 'client'
          ? {
              headers: Headers;
            }
          : {}));
```

そのため、この部分を node_modules 内のコードから削除すると、以下のように補完が効くことが確認できます。
![Untitled](/images/ts-rest-and-jest/Untitled1.png)
なので、ここを修正されれば問題なく使用できそうです。
しかし、そのことについて記載された[イシュー](https://github.com/ts-rest/ts-rest/issues/446)では以下のように回答がありました。

> I think the Nest recommended way to test controllers is outlined here:
>
> [https://docs.nestjs.com/fundamentals/testing#end-to-end-testing](https://docs.nestjs.com/fundamentals/testing#end-to-end-testing)
>
> My company uses a hybrid approach of incorporating unit tests with supertest (instead of full e2e). We don't necessarily rely on the type inference from the controllers, since that is sort of abstracted away (it'll be hard to get the types, like you show here). Here is some sudo code from one of our tests:
>
> …中略
>
> Really, because the controller itself provides compiler type safety, the tests (IMO) should be testing underlying logic that is hard to replicate purely through types.

意訳すると、supertest を使って実際のリクエストに沿った形でテストする方針なので、必ずしも型は必要ないと考えているとのことです。
このことから、ts-rest を使っている場合私が書いたようなテストで body プロパティに型補完をつけるのが厳しいことがわかります。
意図は分かりますが、個人的には型補完が欲しいと思っています。
例で出した簡単な戻り値のテストならいいですが、これが相当数のプロパティを持つオブジェクトであったり、配列であったりするとチェックするための値を用意するのが大変になります。
なので、特にテストしたい一部の値を取り出すなどを想定していましたが、ここまでの流れでそれは厳しそうです。
よって、ts-rest で実装をしていくのは厳しいと思い、断念しました。

## と、思っていたのですが…

先程型補完がきかないから、使用するのを断念すると言いました。
とはいえ、Zod によるリクエストの検証を行ってくれて、別途実装する必要がないのはかなり魅力的です。
なので、どうにかいい感じに使えないかなと思って、色々確認したら許容範囲の対応方法が色々ありましたので、共有します。

### 対応策 ① アサーションをつける

まずはアサーションをつける方法を紹介します。
これは body プロパティを取得する時に、以下のようにアサーションをつける方法です。

```tsx
it("値が取得できること", async () => {
  const result = await appController.getHello();
  const data = await result({ headers: {} });
  const checkVal: z.infer<(typeof c.hello.responses)[200]> = { name: "Hello" };
  const responseBody = data.body as z.infer<(typeof c.hello.responses)[200]>;
  expect(responseBody.name).toEqual("Hello");
  // or
  expect(responseBody).toEqual(checkVal);
});
```

型の自体は ts-rest で API の仕様を決める時に定義しているので、それの body 部分を`z.infer<typeof c.hello.responses[200]>`で型を取り出すだけです。
明示的に型をつけるだけで、この後アプリケーションで使用するわけでもありません。
また、もし定義する型が間違っていてもテストが落ちるので、間違いに気付くこともできます。
なので、ここでアサーション使う選択はありだと思っています。

### 対応策 ② node_modules 内の型定義部分を削除する

もう一つの対応方法は、問題となっている部分のコードを消してテストを実装することです。
[libs/ts-rest/core/src/lib/infer-types.ts](https://github.com/ts-rest/ts-rest/blob/main/libs/ts-rest/core/src/lib/infer-types.ts#L85)で対象部分を削除すれば、補完が効くことは確認しています。
なら、その部分を削除してテストコードを実装すると当初の要望をかなえつつテストができます。
ただ、通常通り書くと node_modules が元に戻ったときにエラーが発生します。
そのため、今回の方法をとったときに以下のように鍵括弧でプロパティをアクセスする必要はあります。

```tsx
expect(responseBody["name"]).toEqual("Hello");
```

こうすれば仮にコードが元に戻ってもエラーになりません。
さらに型が定義されている場合、鍵括弧でも補完は効きますのでテストコードは鍵括弧で書くという取り決めさえしておけば特に問題なく実装できるかと思います。
以上二つの方法は完璧ではないかもしれないですが、個人的には許容できる範囲かなと思っています。
なので、ts-rest でテストはできそうだとは思い、断念するほどではないのかなと思っています。

## OpenAPI と ts-rest どっちを採用するか悩ましい

実は対応策を見つけ出す前に、[Swagger](https://swagger.io/)と[orval](https://orval.dev/)を使って REST API の構築ができていました。
なので、やりたいことはどちらの方法でも行うことができます。
そのため、どちらにするか決める必要があります。
ただ、その決定が中々できません。
OpenAPI の方は余計な関数を挟む必要がないので、テストコードが書きやすいですし、型の補完なども特に気にする必要がありません。
テストが書きやすいので、ロジックのミスが起きにくいのかなと思います。
一方で、ts-rest は定義自体が一つにまとまっており、確認がしやすいのと不正なプロパティが含まれたリクエストを Zod の機能を使うことで自動的にはじいてくれます。
簡単な実装ながらも、テスト以外は痒い所に手が届くことが多いので機能自体は ts-rest が良さそうです。
よって、どちらを使って開発するかが悩みどころです。
正直決めあぐねているので、こっちのほうが良いよがあれば是非ともコメントで教えて欲しいです。
よろしくお願いします。

## おわりに

今回は ts-rest について軽く見た後に、テストコードの時に補完が効かないことや、対応策などをみていきました。
色々一長一短あって、これなら安心・安全というのはないのだなと感じています。
ただ、あまりここで悩みすぎるのも開発が進まないので、早く決めようと思います。
最悪、実装する内容はほぼ一緒なので二つの方式で開発しようとかなと思っています。
流石に無駄ですかね？
ここまで読んでいただきありがとうございました。

## 2024/08/05 追記

この記事では、テストコードの時は型補完が上手くできないという結論で終わっていました。
しかし、フェルリンさんから status の値が 200 だと確定できれば、body 属性の型が決まること教えていただきました。([参照コメント](https://zenn.dev/link/comments/e69d40eece007b))
なので、ts-rest でコントローラーのテストをしたいときは、私が展開した内容ではなく、いただいたコメントのように status を確定させてから値の比較をする方が良いと思います。
フェルリンさん、教えていただきありがとうございました！
