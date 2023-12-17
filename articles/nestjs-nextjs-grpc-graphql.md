---
title: "細けぇことはいいからNestJSとNext.jsで、frontend←(GraphQL)→BFF←(gRPC)→backendを構築する"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NestJS", "nextjs", "GraphQL", "gRPC", "Typescript"]
published: true
---

## はじめに

最近よく耳にする gRPC や GraphQL。
ずっと REST API でやってきた私にとっては、試したいと思いつつ二の足を踏んでいました。
しかし、近々でこれまで親しんできた REST API での実装から離れ gRPC や GraphQL を使った実装を行わないといけない状況がやってくることになりました。
何も心構えをしていないと、実際開発する時にかなり苦労するなと思ったので、一旦触ってみようと思い、この記事をかきました。
ただ、各概念の理解は全然できていないので、今回はとりあえず動かせるようにすることのみを目的とし、概念についてはとりあえず後回しにしています。

## gRPC

まず NestJS でマイクロサービスを使うために`npm i --save @nestjs/microservices`でモジュールをインストールします
その後`npm i --save @grpc/grpc-js @grpc/proto-loader`で gRPC を使うためのモジュールをインストールします。
インストールが完了したら、main.ts を以下のように変更します。

```jsx
import { NestFactory } from '@nestjs/core';
import { Transport, MicroserviceOptions } from '@nestjs/microservices';
import { AppModule } from './app.module';
import * as path from 'path';
async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.GRPC,
      options: {
        package: 'hero',
        protoPath: path.join(__dirname, 'hero/hero.proto'),
      },
    },
  );
  await app.listen();
}
bootstrap();
```

そして、nest-cli.json に以下のコードを compilerOptions に記載し、dist に proto ファイルがコピーされるようにします。

```jsx
"assets": ["**/*.proto"],
"watchAssets": true
```

現状 proto ファイルを Typescript でいい感じに使えることができないので、`npm install ts-proto`を実行して、proto ファイルを Typescript に変換してくれるモジュールをインストールします。
ここまで下準備が完了です。
次に、nest コマンドで hero フォルダ配下に hero.controller.ts と hero.module.ts を準備しておきます。
そして、hero フォルダ配下に以下の proto ファイルを作成します。

```jsx
syntax = "proto3";
package hero;
service HeroesService {
  rpc FindOne (HeroById) returns (Hero) {}
}
message HeroById {
  int32 id = 1;
}
message Hero {
  int32 id = 1;
  string name = 2;
}
```

その後、`npx protoc --ts_proto_opt=nestJs=true --plugin=./node_modules/.bin/protoc-gen-ts_proto --ts_proto_out=. ./src/hero/hero.proto`で proto ファイルを Typescript に変換します。
変換後には以下の hero.ts が生成されています。

```jsx
/* eslint-disable */
import { GrpcMethod, GrpcStreamMethod } from "@nestjs/microservices";
import { Observable } from "rxjs";
export const protobufPackage = "hero";
/** hero/hero.proto */
export interface HeroById {
  id: number;
}
export interface Hero {
  id: number;
  name: string;
}
export const HERO_PACKAGE_NAME = "hero";
export interface HeroesServiceClient {
  findOne(request: HeroById): Observable<Hero>;
}
export interface HeroesServiceController {
  findOne(request: HeroById): Promise<Hero> | Observable<Hero> | Hero;
}
export function HeroesServiceControllerMethods() {
  return function (constructor: Function) {
    const grpcMethods: string[] = ["findOne"];
    for (const method of grpcMethods) {
      const descriptor: any = Reflect.getOwnPropertyDescriptor(
        constructor.prototype,
        method
      );
      GrpcMethod("HeroesService", method)(
        constructor.prototype[method],
        method,
        descriptor
      );
    }
    const grpcStreamMethods: string[] = [];
    for (const method of grpcStreamMethods) {
      const descriptor: any = Reflect.getOwnPropertyDescriptor(
        constructor.prototype,
        method
      );
      GrpcStreamMethod("HeroesService", method)(
        constructor.prototype[method],
        method,
        descriptor
      );
    }
  };
}
export const HEROES_SERVICE_NAME = "HeroesService";
```

そうしたら、hero.controller.ts を以下のように変更します。

```jsx
import { Controller } from "@nestjs/common";
import { GrpcMethod } from "@nestjs/microservices";
import { HEROES_SERVICE_NAME, Hero, HeroById } from "./hero";
import { Metadata, ServerUnaryCall } from "@grpc/grpc-js";
@Controller()
export class HeroesController {
  @GrpcMethod(HEROES_SERVICE_NAME)
  findOne(
    data: HeroById,
    metadata: Metadata,
    call: ServerUnaryCall<any, any>
  ): Hero {
    const items = [
      { id: 1, name: "John" },
      { id: 2, name: "Doe" },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

これで gRPC 側の準備が完了しました。
では、BFF に入る前に動作確認をします。
そのまま NestJS を起動してもアクセスはできないので、[evans](https://github.com/ktr0731/evans)を使います。
`curl -OL https://github.com/ktr0731/evans/releases/download/v0.10.11/evans_linux_amd64.tar.gz`を実行して、`tar -zxvf evans_linux_amd64.tar.gz`でファイルを解凍します。
その後、`sudo mv evans /usr/bin`を実行して、パスを通します。
`evans --host localhost -p 5000 src/hero/hero.proto`で evans ターミナルを立ち上げます。
![2023-12-17_00h26_04.png](/images/nestjs-nextjs-grpc-graphql/2023-12-17_00h26_04.png)
画像のようなターミナルが起動すれば成功です。
call FindOne を入力して、実行すると id の入力を求められるので、1 を入力すると以下のデータが表示されれば成功です。

```json
{
  "id": 1,
  "name": "John"
}
```

## BFF

次に BFF を実装します。
実装始める前に注意点ですが、この後のフロントエンドを実装する際の兼ねあいで、main.ts のポート番号は 3000 から 3003 に変更しています。
その点はご認識お願いします。
では実装を始めます。
まずは先程作成した、gRPC を呼び出せるようにします。
gRPC の時と同様に`npm i --save @nestjs/microservices`でマイクロサービス用のモジュールをインストールします。
そして、gRPC で作成したのと同じ hero.proto ファイルを作成します。
`npm install ts-proto`で ts-proto をインストールして、`npx protoc --ts_proto_opt=nestJs=true --plugin=./node_modules/.bin/protoc-gen-ts_proto --ts_proto_out=. ./src/hero/hero.proto`を実行し proto から Typescript ファイルを作成します。
hero.ts の作成を確認できたら、app.module.ts を以下のように変更します。

```tsx
import { Module } from "@nestjs/common";
import { AppService } from "./app.service";
import { ClientsModule, Transport } from "@nestjs/microservices";
import { HERO_PACKAGE_NAME } from "./hero/hero";
import * as path from "path";
import { AppResolver } from "./app.resolver";
import { AppController } from "./app.controller";
@Module({
  imports: [
    ClientsModule.register([
      {
        name: HERO_PACKAGE_NAME,
        transport: Transport.GRPC,
        options: {
          url: "localhost:5000",
          package: HERO_PACKAGE_NAME,
          protoPath: path.join(__dirname, "hero/hero.proto"),
        },
      },
    ]),
  ],
  controllers: [AppController],
  providers: [AppService, AppResolver],
})
export class AppModule {}
```

gRPC で定義したサービスを呼び出す準備ができたので、app.service.ts を以下のように変更して、proto で定義したサービスをプロパティに格納します。

```tsx
import { Inject, Injectable, OnModuleInit } from "@nestjs/common";
import {
  HEROES_SERVICE_NAME,
  HERO_PACKAGE_NAME,
  Hero,
  HeroesServiceClient,
} from "./hero/hero";
import { ClientGrpc } from "@nestjs/microservices";
import { Observable, lastValueFrom } from "rxjs";
import { adjustRpcResponse } from "./utils/convertObservableToPromise";

@Injectable()
export class AppService implements OnModuleInit {
  private heroesService: HeroesServiceClient;

  constructor(@Inject(HERO_PACKAGE_NAME) private client: ClientGrpc) {}

  onModuleInit() {
    this.heroesService =
      this.client.getService<HeroesServiceClient>(HEROES_SERVICE_NAME);
  }

  async getHero() {
    return await this.adjustRpcResponse<Hero>(
      this.heroesService.findOne({ id: 1 })
    );
  }

  private async adjustRpcResponse<T>(response: Observable<T>): Promise<T> {
    try {
      return await lastValueFrom(response);
    } catch (e) {
      throw e;
    }
  }
}
```

これで、gRPC のサービスを BFF 側でも呼び出せるようになりました。
実際に確認してみます。
app.controller.ts を以下のように変更して、npm run start で NestJS を起動します。

```tsx
import { Controller, Get } from "@nestjs/common";
import { Observable } from "rxjs";
import { AppService } from "./app.service";
import { Hero } from "./hero/hero";
@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}
  @Get()
  call(): Observable<Hero> {
    return this.appService.getHero();
  }
}
```

そして、http://localhost:3000 にアクセスすると、先程確認した JSON が表示されます。

```json
{
  "id": 1,
  "name": "John"
}
```

ここまでで、BFF と gRPC の通信ができました。
次にフロント側との通信を行うために、GraphQL を準備します。
GraphQL を使えるようにするために、`npm i @nestjs/graphql @nestjs/apollo @apollo/server graphql`を実行して、必要なモジュールをインストールします。
インストールしたら、GraphQL についての機能を使用するために、app.module.ts を変更します。

```tsx
import { Module } from "@nestjs/common";
import { AppService } from "./app.service";
import { ClientsModule, Transport } from "@nestjs/microservices";
import { HERO_PACKAGE_NAME } from "./hero/hero";
import * as path from "path";
import { AppResolver } from "./app.resolver";
import { AppController } from "./app.controller";
import { GraphQLModule } from "@nestjs/graphql";
import { ApolloDriver, ApolloDriverConfig } from "@nestjs/apollo";
@Module({
  imports: [
    ClientsModule.register([
      {
        name: HERO_PACKAGE_NAME,
        transport: Transport.GRPC,
        options: {
          url: "localhost:5000",
          package: HERO_PACKAGE_NAME,
          protoPath: path.join(__dirname, "hero/hero.proto"),
        },
      },
    ]),
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: path.join(process.cwd(), "src/schema.gql"),
    }),
  ],
  controllers: [AppController],
  providers: [AppService, AppResolver],
})
export class AppModule {}
```

コードファーストで実装する場合は、オプションに autoSchemaFile プロパティを設定する必要があります。([ドキュメント](https://docs.nestjs.com/graphql/quick-start#code-first))
autoSchemaFile プロパティについては、スキーマファイルを生成したい場合は、作成して欲しいファイルパスを指定します。
一方で、スキーマだけ作りスキーマファイルが不要な場合は、autoSchemaFile プロパティの値を true にします。
これによって、メモリのみにスキーマが保存されている状態となります。
リゾルバーの戻り値を定義するためのモデルを作成するために、app.model.ts を作り以下のコードを実装します。

```tsx
import { Field, ObjectType } from "@nestjs/graphql";

@ObjectType()
export class AppModel {
  @Field((type) => Number)
  id: number;
  @Field((type) => String)
  name: string;
}
```

app.resolver.ts を作成し、以下のようにリゾルバーを実装します。

```tsx
import { Query, Resolver } from "@nestjs/graphql";
import { AppModel } from "./app.model";
import { AppService } from "./app.service";
@Resolver((of) => AppModel)
export class AppResolver {
  constructor(private appService: AppService) {}
  @Query(() => AppModel, { name: "apps" })
  async getHero(): Promise<AppModel> {
    const hero = await this.appService.getHero();
    return { id: hero.id, name: hero.name };
  }
}
```

## フロント

まずは Next.js プロジェクトを`npm create next-app frontend --typescript`で作成してください。
聞かれる内容は、任意のもので構いませんが、今記事ではフォルダを src 配下にまとめていないのと、App Router ではなく Page Router を使用しています。
App Router よくわかりませんでした…。
Next.js プロジェクトを作成したら、GraphQL でデータを取得するため、[urql のドキュメント](https://formidable.com/open-source/urql/docs/basics/react-preact/)に従って`npm install --save urql`で urql モジュールをインストールします。
urql をインストールしたら、\_app.tsx を以下のように変更します。

```tsx
import "@/styles/globals.css";
import type { AppProps } from "next/app";
import { Client, Provider, cacheExchange, fetchExchange } from "urql";
const client = new Client({
  url: "http://localhost:3000/graphql",
  exchanges: [cacheExchange, fetchExchange],
});
export default function App({ Component, pageProps }: AppProps) {
  return (
    <Provider value={client}>
      <Component {...pageProps} />
    </Provider>
  );
}
```

これで、urql の useQuery が使用できるようになりました。
そしたら、index.tsx を以下のようにクエリを使ってデータを取るようにします。

```tsx
import { Inter } from "next/font/google";
import { gql, useQuery } from "urql";
const inter = Inter({ subsets: ["latin"] });
const heroQuery = gql`
  query heros {
    apps {
      id
      name
    }
  }
`;
const Home = () => {
  const [{ data }] = useQuery({ query: heroQuery });
  return (
    <main
      className={`flex min-h-screen flex-col items-center justify-between p-24 ${inter.className}`}
    >
      <div className="z-10 max-w-5xl w-full items-center justify-between font-mono text-sm lg:flex">
        <p>id:{data?.apps.id}</p>
        <p>name:{data?.apps.name}</p>
      </div>
    </main>
  );
};
export default Home;
```

では、`npm run dev`でアプリケーション起動して動作確認をしたいところですが、現状は CORS エラーによって、BFF を呼び出すことができません。
そのため、リバースプロキシ設定を行います。
next.config.js を以下のように、rewrites 関数を追加します。

```tsx
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  async rewrites() {
    return [
      {
        source: "/graphql",
        destination: `http://localhost:3003/graphql`,
      },
    ];
  },
};
module.exports = nextConfig;
```

これを設定すれば、http://localhost:3000/graphql へリクエストした時に、自動で BFF 側へリクエストを飛ばしてくれるようになります。
なので、CORS エラーを回避することができます。
これで、ようやく画面での確認ができるようになりました。
`npm run dev`でアプリケーションを起動し、http://localhost:3000 にアクセスすると、以下の画面が表示されます。
![2023-12-17_11h45_43.png](/images/nestjs-nextjs-grpc-graphql/2023-12-17_11h45_43.png)
以上で完了と言いたいところですが、このままで終わるには看過できない問題が残っています。
それは、以下の二つです。
① クエリの定義が間違っていても、動作確認をするまで分からない。
②useQuery の戻り値が any 型である。
動作確認をするまで分からないというのは、Typescript を使っている意味がないですし、何より開発体験がよくありません。
そこで、[GraphQL Code Generator](https://the-guild.dev/graphql/codegen)を使い、誤ったクエリの時はエラーを発生させるようにし、クエリに問題が無ければ戻り値の型定義ファイルを生成させるようにします。
[ドキュメント](https://the-guild.dev/graphql/codegen/docs/guides/react-vue)に従い、以下のコマンドで必要なモジュールをインストールします。

```
npm i -S graphql
npm i -D typescript ts-node @graphql-codegen/cli
```

そしたら、`npx graphql-codegen init`で初期設定を行います。
その際に以下のような質問があるので、適宜入力してください。

```
質問①
? What type of application are you building? (Use arrow keys):(アプリケーション種類は何？)
→「Application built with React 」を選択します。

質問②
? Where is your schema?:(スキーマの定義はどこにありますか)
→スキーマファイルのパス(今回はbff内のsrc配下にあるschema.gql) or スキーマにアクセスできるURL(今回はhttp://localhot:3003/graphql)

質問③
Where are your operations and fragments?:(GraphQLのクエリなどを書いているファイルはどこにありますか？)
→GraphQLのクエリなどが記載されているパスを指定します。この記事では「pages/**/*.tsx」を指定します。

質問④
Where to write the output:(どこにコードを生成する？)
→任意のディレクトリを設定してください。この記事では「generate/」を指定しています。
→ディレクトリの最後に「/」を設定しないとコード生成時にエラーが発生するので、ご注意ください。

質問⑤
Do you want to generate an introspection file?
→これの意味はよく分からなかったので、好きなものを選んでください。

質問⑥
How to name the config file?:(コード生成の設定ファイル名は何にする？)
→ymlも行けますが、特にこだわりが無ければデフォルトのままで良いと思います。

質問⑦
What script in package.json should run the codegen?:(package.jsonでコード生成を起動するスクリプト名は何にする?)
→好きなものを選んでください。
```

なお、質問 ④ については検索するとファイル名を指定しているものが結構ヒットします。
ただ、今回のコード生成は preset に client を指定しているため、ファイル名を設定するとエラーが発生しコード生成ができません。
なので、必ずディレクトリのパスを指定してください。
preset の client についての説明は、私自身まだ理解ができていないので、[ドキュメント](https://the-guild.dev/graphql/codegen/plugins/presets/preset-client)や[こちらの記事](https://zenn.dev/mh4gf/articles/graphql-codegen-client-preset)に丸投げします。
これでコード生成の準備ができました。
実際に、`npm run codegen`を実行してみます。
すると、generate ディレクトリの中に何やら、戻り値の型定義やクエリを呼び出すための文字列が記載されている graphql 関数などを見つけることができます。
また、試しにクエリ部分の name を title に変更して、再度`npm run codegen`を実行してみてください。
スキーマに存在しないから、コード生成ができないというエラーが発生すると思います。
これでコード生成ができたので、実際に使ってみます。
では、先程の index.tsx を以下のように修正します。

```tsx
import { graphql } from "@/generate";
import { HerosQuery } from "@/generate/graphql";
import { Inter } from "next/font/google";
import { useQuery } from "urql";
const inter = Inter({ subsets: ["latin"] });
const herosDocument = graphql(`
  query heros {
    apps {
      id
      name
    }
  }
`);
const Home = () => {
  const [{ data }] = useQuery<HerosQuery>({ query: herosDocument });
  return (
    <main
      className={`flex min-h-screen flex-col items-center justify-between p-24 ${inter.className}`}
    >
      <div className="z-10 max-w-5xl w-full items-center justify-between font-mono text-sm lg:flex">
        <p>id:{data?.apps.id}</p>
        <p>name:{data?.apps.name}</p>
      </div>
    </main>
  );
};
export default Home;
```

useQuery に型を付与できるようになり、補完が効くことが確認できると思います。
ここまでで、コードの自動生成ができるようになり、クエリのチェックや戻り値の型定義を取得できるようになりました。
ただ、現状だと実装して、コードの生成コマンドを叩いて反映されたら修正してと、手間がかかっています。
なので、graphql-codegen の watch モードを有効にして、クエリなどの変更があれば自動で再生成を行うようにします。
wathc モードを有効にするために、`npm i @parcel/watcher`で@parcel/watcher モジュールをインストールします。
後は`npm run codegen --watch`を実行すれば、変更検知してくれるようになります。

## 今後の課題？

### ① クエリの補完について

フロント側でクエリを書く際に補完を効かせることはできるのかが分からない状態です。
BFF で playground を起動して、そこで色々試した結果をフロント側に転記するしかないのか、Next.js 側でいい感じにクエリの補完が効くのかが不明でした。
[こちらの記事](https://techlife.cookpad.com/entry/2021/03/24/123214)を参照するに、存在しえないクエリは書けないので、それで対応するしかないのかと思いつつ、コード生成の前に静的なエラーを出してくれないものかと思っています。

### ②proto ファイルの二重管理

今回、バックエンドと BFF で同一の proto ファイルを作成しています。
明らかに今後ミスが起きる状態となっているので、一つの proto で BFF、バックエンド側両方に Typescript ファイルを生成できるようにする必要があります。

### ③ モノレポ？何それ？からの脱却

全て別プロジェクトで動かしたので、モノレポなりで単一管理ができるようにする必要があると感じています。
モノレポで管理できたら、「②proto ファイルの二重管理」もついでに解消できないかなと淡い期待をしています。

### ④GraphQL、gRPC よくわからない

GraphQL については、Resolver？MVC モデルの controller みたいなもの？、Mutation?Post や Put メソッドどこ行った？という状態です。
gRPC については、Stream?、複数リクエストに対して一つのレスポンスを返すとは???、双方向 Stream????????という状態です。
なので、そもそもの概念について理解する必要があると思っています。

## おわりに

今回はとりあえず frontend <- (GraphQL) -> BFF <- (gRPC) -> backend の流れを構築しました。
この機能は何をしているかといった概念が分からないとか、proto の管理が微妙などまだまだ全然分かっていないことがたくさんあります。
しかし、とりあえずざっくりとした全体像はつかめたかもしれないので、これからやって来る gRPC や GraphQL の波に乗るための板は準備できたと思います。
準備ができただけなので、波を乗りこなせるように引き続き学習はしていきます。
ここまで読んでいただきありがとうございました。

## 参考資料

gRPC 周り
[https://zenn.dev/optimisuke/articles/b44f51311c1789](https://zenn.dev/optimisuke/articles/b44f51311c1789)
[https://zenn.dev/daimyo404/articles/710c61fd6aa877](https://zenn.dev/daimyo404/articles/710c61fd6aa877)
[https://docs.nestjs.com/microservices/grpc](https://docs.nestjs.com/microservices/grpc)
BFF 周り
[https://zenn.dev/aoito/articles/07cd081ae7771e](https://zenn.dev/aoito/articles/07cd081ae7771e)
evans 導入周り
[https://zenn.dev/kumamoto/articles/90e0dbddb02c9c#3.-evans](https://zenn.dev/kumamoto/articles/90e0dbddb02c9c#3.-evans)
[https://github.com/ktr0731/evans/releases](https://github.com/ktr0731/evans/releases)
NestJS で GraphQL を動かすための準備
[https://zenn.dev/fjsh/articles/nestjs-graphql](https://zenn.dev/fjsh/articles/nestjs-graphql)
[https://docs.nestjs.com/graphql/quick-start](https://docs.nestjs.com/graphql/quick-start)
Observal を Promise に変換する時に役立ったもの
[https://blog.nnn.dev/entry/2023/10/13/110000](https://blog.nnn.dev/entry/2023/10/13/110000)
curl コマンドを使ったファイルダウンロードについて
[https://qiita.com/ponsuke0531/items/38e6421058364b6f8562](https://qiita.com/ponsuke0531/items/38e6421058364b6f8562)
Next.js のリバースプロキシ設定
[https://qiita.com/zksytmkn/items/884a9e97b59ceca846a9](https://qiita.com/zksytmkn/items/884a9e97b59ceca846a9)
コード自動生成
[https://zenn.dev/monicle/articles/e427d17935d019](https://zenn.dev/monicle/articles/e427d17935d019)
[preset が client の時の動作理解に役立った](https://zenn.dev/mh4gf/articles/graphql-codegen-client-preset)
コード自動生成の wathc モードを有効にする([@parcel/watcher](https://www.npmjs.com/package/@parcel/watcher)インストール必須)
[https://the-guild.dev/graphql/codegen/docs/getting-started/development-workflow#watch-mode](https://the-guild.dev/graphql/codegen/docs/getting-started/development-workflow#watch-mode)
