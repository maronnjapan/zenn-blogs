---
title: "Prismaでデータベースを複数作成して、PrismaClientをモックせずにテストを実行する"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["prisma", "typescript"]
published: true
---

## はじめに

アプリケーションを開発する際、何かしらのデータベースを使わないことをはほぼないと言えます。
ならば、テストコードを記載する際にもデータベースのことを無視することはできません。
そのため、世の中にはテスト用にデータベースを操作できるモジュールが多く存在しています。
しかし、それらモジュールはアプリケーションで使用しているモジュールそのものではありません。
基本的には互換性を持たせるようになっていますが、ORM に機能が追加された時など即時で反映されないものはテストができなくなってしまいます。
さらに、データベースが一つだとアプリケーション側で作成したデータがテストに影響を及ぼす可能性も高いです。
そこで、いっそのことデータベースを操作するのはアプリケーションで使用しているモジュールと同様のものを用いて、テストの時はデータベースの接続先を変更しようと思ったのが、今回書いた動機です。
具体的な手順は以下の通りです。
まず、devcontainer を使用して構築したローカル環境で複数のデータベースを作成します。
その後、作成した複数のデータベースにそれぞれ紐づいたスキーマファイルを生成します。
最後に各スキーマに合わせた PrismaClient を作成し、テスト実行時はテスト用の PrismaClient を使用できるようにします。
なお、今回は devcontainer で行っていますが、記載する場所が docker-compose.yml なので、devcontainer を使用していなくても同じようなことはできると思います。
また、この記事は以下の 4 点が前提条件として存在しているので、ご認識お願いします。

- 使用しているデータベースは PostgreSQL
- Postgre のコンテナを作成する docker-compose.yml が.devcontainer ディレクトリに存在する
- アプリケーションは NestJS で動いている
- アプリケーションに Prisma を使用するためのモジュールはインストールされており、初期設定は完了している。

それでは始めます。

## データベースを複数作成する

### データベースを複数作成する SQL を作成

まずは、データベースを複数作成するための SQL ファイルを作成します。
といっても、SQL ファイル自体はとても単純で以下のようなファイルを作成すれば問題ありません。

```sql
CREATE DATABASE postgres;
CREATE DATABASE postgres_test;
```

なので、.devcontainer ディレクトリ内に postgre-init ディレクトリを作成し、その中に上記 SQL ファイルを作成します。
これでデータベースを複数作成する準備ができたので、これを動かせるように docker-compose.yml を編集します。

### docker-compose.yml で作成した SQL ファイルを実行する

[Postgre の docker イメージについてのドキュメント](https://hub.docker.com/_/postgres#:~:text=and%20POSTGRES_DB.-,Initialization%20scripts,-If%20you%20would)を確認すると、初期設定を行う時に docker-entrypoint-initdb.d ディレクトリに`*.sql`, `*.sql.gz`, `*.sh`のいずれかのファイルが存在すれば、そのファイルを実行するとあります。
なので、先程作成した SQL ファイルを docker-entrypoint-initdb.d へ配置できりように、.devcontainer/docker-compose.yml の volumes に以下の記載を追加します。

```yaml
- ./postgres-init:/docker-entrypoint-initdb.d
```

なお、注意点として docker-entrypoint-initdb.d 内のファイルが実行されるのは、data ディレクトリにデータが存在しない時のみです。
そのため、実行する SQL ファイルなどを変更したいときは、volume を削除して data ディレクトリの中を空にする必要があります。
volume を削除するので、データベースなどは全て消えてしまいます。
なので、消したくないデータはバックアップを取っておくなどの対処をしてから行う必要があります。
ここまでで、データベースを複数作成できるようになりました。
では、実際にコンテナを起動して、ターミナルで`docker exec -it PostgreSQLのコンテナ名 or コンテナID psql -U postgres`を実行します。
コンテナ内に入るので、その中で\l コマンドを実行しデータベース一覧を確認すると、postgre データベースと postgre_test データベースが存在することを確認できます。
では、次に NestJS でそれぞれのデータベースへ接続できるようにします。

### Prisma で複数のデータベースに接続する

ここでは、Prisma で複数のデータベースに接続できているかを Prisma Studio を使って確認します。
まず、作成したデータベースに接続するための URL を.evn ファイルに設定します。

```
DATABASE_URL="postgresql://postgres:postgres@db:5432/postgres?schema=public"
DATABASE_TEST_URL="postgresql://postgres:postgres@db:5432/postgres_test?schema=public"
```

その後、prisma ディレクトリ内にある prisma.schema を以下のように記載します。

```yaml
generator client {
provider = "prisma-client-js"
}
datasource db {
provider = "postgresql"
url      = env("DATABASE_URL")
}
model User {
id       String @id @default(uuid())
email    String
password String
}
```

これで、`npx prisma db push`を実行します。
その後、`npx prisma studio`を実行し、ターミナルに表示される URL にアクセスします。
すると Prisma Studio の画面が表示されるので、User テーブルを選択すると以下の画面が表示されます。
![2023-12-24_10h43_01.png](/images/prisma-databases-for-test/2023-12-24_10h43_01.png)
上にある「Add Record」をクリックし、任意のデータを作成し保存します。
Prisma Studio を閉じて、schema.prisma の`DATABASE_URL`を`DATABASE_TEST_URL`に変更して、`npx prisma db push`を実行します。
再度`npx prisma studio`を実行し、User テーブルを確認すると先程作成したデータが存在していません。
これは今見ているデータベースが postgre_test テーブルになっているためです。
実際、`DATABASE_TEST_URL`を`DATABASE_URL`に戻し、再度 push した後 Prisma Studio にアクセスすると先程作成したデータが存在することを確認できます。
以上で、Prisma を使って複数のデータベースにアクセスできるようになりました。
次にアプリケーション用の PrismaService とテスト用の PrismaService を作成します。

### 2 種類の PrismaService を作成する

src 配下の任意のディレクトリで以下の Prisma を呼び出すファイルを 2 つ作成します。
なお、今回私は間違い防止のために、prisma ディレクトリにアプリケーション用の prisma.service.ts を作成し、prisma-test ディレクトリに prisma-test.service.ts を作成しております。
まずはテスト用の PrismaService と PrismaModule です。

```tsx
// prisma-test/prisma-test.service.ts
import { Injectable } from "@nestjs/common";
import { PrismaClient } from "@prisma/client";
const { DATABASE_TEST_URL } = process.env;
@Injectable()
export class PrismaTestService extends PrismaClient {
  constructor() {
    super({ datasources: { db: { url: DATABASE_TEST_URL } } });
  }
}
```

```tsx
// prisma-test/prisma-test.module.ts
import { Module } from "@nestjs/common";
import { PrismaTestService } from "./prisma-test.service";
@Module({
  providers: [PrismaTestService],
  exports: [PrismaTestService],
})
export class PrismaTestModule {}
```

次にアプリケーションで使用する PrismaService と PrismaModule です。

```tsx
// prisma/prisma.service.ts
import { Injectable } from "@nestjs/common";
import { PrismaClient } from "@prisma/client";
const { DATABASE_URL } = process.env;
@Injectable()
export class PrismaTestService extends PrismaClient {
  constructor() {
    super({ datasources: { db: { url: DATABASE_URL } } });
  }
}
```

```tsx
// prisma/prisma.module.ts
import { Module } from "@nestjs/common";
import { PrismaService } from "./prisma.service";
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

どちらも環境変数から、データベースを接続するための URL を取得し、それをコンストラクタで明示的に接続先として指定しています。
これによって、それぞれの PrismaService は機能そのものは Prisma を元にしているが、見ているデータベースの向き先が異なるので、アプリケーション用とテスト用で分けることができます。
これで準備ができたので、動作確認をします。
まずはアプリケーション側で確認します。
app.module.ts で`imports: [PrismaModule, PrismaTestModule],`を記載したら、app.controller.ts を以下のように変更します。

```tsx
import { Controller, Get } from "@nestjs/common";
import { PrismaService } from "./prisma/prisma.service";
@Controller()
export class AppController {
  constructor(private readonly prismaService: PrismaService) {}
  @Get("/user/test")
  async getUsers() {
    return await this.prismaService.user.findMany();
  }
}
```

これで、`npm run start`を実行し、http://localhost:3000/user/test にアクセスすると先程作成したユーザーデータが表示されていることが確認できます。
一方、PrismaService を PrismaTestService に変更して、http://localhost:3000/user/test に再度アクセスすると空配列が返ってくることを確認できます。
アプリケーション側は切り替えができることを確認できたので、次にテストでも切り替えができるかを見ていきます。
app.controller.ts でアプリケーション用の PrismaService に変更したら、app.controller.spec.ts を作成し、以下のファイルを記載します。

```tsx
import { Test, TestingModule } from "@nestjs/testing";
import { AppController } from "./app.controller";
import { PrismaService } from "./prisma/prisma.service";
describe("AppController", () => {
  let appController: AppController;
  beforeEach(async () => {
    const app: TestingModule = await Test.createTestingModule({
      controllers: [AppController],
      providers: [PrismaService],
    }).compile();
    appController = app.get<AppController>(AppController);
  });
  it("ユーザーを取得できること", async () => {
    const result = await appController.getUsers();
    expect(result).toHaveLength(1);
  });
});
```

`npm run test -- app.controller.spec.ts`でテストを実行すると、テストが通ることを確認できます。
次にテスト用の PrismaService を使用するために、テストコード内の providers 部分を以下のように変更します。

```tsx
providers: [{
	provide: PrismaService,
	useClass: PrismaTestService
}],
```

NestJS が提供する`Test.createTestingModule`は上記のように記載すれば、アプリケーション側で使用しているクラスをテストの際は useClass で指定したクラスに置き換えて実行するようにしてくれます。
では、`npm run test -- app.controller.spec.ts`でテストを実行すると、今度はテストが落ちることを確認できます。
postgre_test データベースにはデータを作成していないので、テストの時は使用している PrismaService がテスト用のものになっているのが分かります。
以上で複数のデータベースを用いて、テスト用とアプリケーション用の PrismaService を用いることができました。
これで、Prisma の機能をラップしたモジュールを用いずともテストができるようになり、モジュールによる制限でテストができなくなるという問題がなくなりました。
しかし、現状はまだ課題が残っています。
それは以下の点です。

1. DB の定義変更をテスト用、アプリケーション用のデータベースに反映させるには、schema.prisma の URL を都度変更して push 必要がある。
2. そのため、テスト用とアプリケーション用のデータベースで定義の差異がおきてしまい、エラーが発生する可能性があるので、開発体験が落ちる。
3. さらに、URL を変更する接続先を間違えた状態でアプリケーションをデプロイしてしまう可能性がある。

そのため、最後に[Prisma Import](https://github.com/ajmnz/prisma-import)を用いて、テスト用とアプリケーション用それぞれのスキーマファイル生成し、一度の script コマンドで両方のデータベースに反映できるようにします。

### Prisma Import を用いたスキーマファイルの生成

ここではテスト用とアプリケーション用のスキーマファイルを作ります。
作り方の方向としては以下の通りです。

- テストとアプリケーションそれぞれの初期設定のみを記載したスキーマファイルを作成
- 同階層にテーブル定義のみをまとめたスキーマファイルを作成
- Prisma Import を用いて上記二種類のスキーマファイルを結合し、テスト用とアプリケーション用として動かせるスキーマファイルを作成
  では始めます。

`npm i -D prisma-import`で Prisma Import をインストールします。
次に prisma ディレクトリ内に任意のディレクトリ(今回は parts)を作成し、それぞれ以下の内容を記載した models.prisma,schema_product.prisma,schema_test.prisma を作成します。

```tsx
//models.prisma
model User {
  id       String @id @default(uuid())
  email    String
  password String
}
```

```tsx
//schema_product.prisma
generator client {
    provider = "prisma-client-js"
}
datasource db {
    provider = "postgresql"
    url      = env("DATABASE_URL")
}
```

```tsx
//schema_test.prisma
generator client {
    provider = "prisma-client-js"
}
datasource db {
    provider = "postgresql"
    url      = env("DATABASE_TEST_URL")
}
```

上記設定が完了したら、package.json に以下の script コマンドを追加します。

```json
"db:push:local": "npx prisma-import -f -s ./prisma/parts/{models,schema_product}.prisma -o ./prisma/schema.prisma && prisma db push && npx prisma-import -f -s ./prisma/parts/{models,schema_test}.prisma -o ./prisma/schema_test.prisma && prisma db push --schema=./prisma/schema_test.prisma"
```

それぞれのコマンドを説明します。
`npx prisma-import -f -s ./prisma/parts/{models,schema_product}.prisma -o ./prisma/schema.prisma`
上記コマンドは prisma/parts 配下にある models.prisma ファイルと schema_product.prisma ファイルを取得し、それらを結合したものを schema.prisma として生成することを意味しています。
f オプションはすでに schema.prisma ファイルが存在しいた場合上書きを強制的に行うオプションです。
s オプションは結合する prisma ファイルを設定するオプションです。ワイルドカードを使用することができ、一致するファイルを全て取得します。
o オプションは prisma ファイルを生成するパスを指定するオプションです。
`prisma db push`
これは馴染みのある、スキーマファイルの内容を DB に反映させるコマンドです。
特にオプションを設定しなければ、schema.prisma の内容を反映させます。
明示的に反映させるスキーマファイルを指定したい場合は`--schema=”スキーマファイルのパス”`を付与して実行すれば、schema.prisma 以外のスキーマファイルを実行します。
以上より、`npx prisma-import -f -s ./prisma/parts/{models,schema_product}.prisma -o ./prisma/schema.prisma && prisma db push`はアプリケーション用のスキーマファイルを生成し、それを反映させるコマンドであると分かります。
上記コマンドの後に記載されている以下のコマンドは、やりたいことはほぼ一緒です。
`npx prisma-import -f -s ./prisma/parts/{models,schema_test}.prisma -o ./prisma/schema_test.prisma && prisma db push --schema=./prisma/schema_test.prisma`
異なるのはテスト用のスキーマファイルを生成し、それをデータベースに反映させている点のみです。
以上のコマンドを用いて開発を行えば、常に同一テーブル定義を保ちつつ、都度接続先を変更せずにテスト用とアプリケーション用のデータベースを使い分けることができます。
これで最初に言及した、反映漏れやうっかりアプリケーション用の URL を間違えたままデプロイしてしまう心配がなくなります。
後はテストファイルでアプリケーション用の PrismaService をテスト用に置き換え忘れを防止するために、prisma.service.ts のコンストラクタに以下のコードを記載します。

```tsx
if (process.argv.some((arg) => arg.includes("jest"))) {
  throw new Error("Can Not Use This Class In Test File");
}
```

以上で、アプリケーション用とテスト用のデータベースを共存させつつ機能の実装とテストの実装ができるようになりました。
これでより実際に即したテストを行いつつ、アプリケーション側の影響を受けないようにできました。

## 余談 テスト後にデータを削除する

モジュールを使わずに自前でデータを作成する場合、データを削除する処理も自前で行う必要があります。
ここでは、その方法について言及します。
といっても、コードは[こちらの記事](https://zenn.dev/seya/articles/0aea06f9eb6815)をほぼ丸パクリしたものとなっています。

```tsx
const app: TestingModule = await Test.createTestingModule({
  controllers: [AppController],
  providers: [
    {
      provide: PrismaService,
      useClass: PrismaTestService,
    },
  ],
}).compile();
appController = app.get<AppController>(AppController);
const prisma = app.get(PrismaService);
const allProperties = Object.keys(prisma);
let modelNames = allProperties.filter(
  (x) => !(typeof x === "string" && (x.startsWith("$") || x.startsWith("_")))
);
for (const model of modelNames) {
  await prisma[model].deleteMany();
}
```

一応やっていることは PrismaClient クラスのプロパティを取得し、テーブル名以外のプロパティを除去した後に、テーブル名を動的に設定し Prisma の機能を使ってデータを消すようにしています。
注意点として、記事内でも記載があるように、リレーションの仕方によってはエラーが発生します。
そのため、リレーションエラーが起きても最終的に全てのデータを消せるようにしたい場合は以下のように記載する方法など別途処理を追加する必要があります。

```tsx
// ...略
const prisma = app.get(PrismaService);
const allProperties = Object.keys(prisma);
let modelNames = allProperties.filter(
  (x) => !(typeof x === "string" && (x.startsWith("$") || x.startsWith("_")))
);
while (modelNames.length > 0) {
  try {
    for (const model of modelNames) {
      await prisma[model].deleteMany();
      modelNames = modelNames.filter((name) => name !== model);
    }
  } catch (e) {}
}
```

まあ、上記コードは二重ループなのであまり良いとは思いません。
もっといい方法があれば是非教えてください。

## おわりに

今回はデータベースを複数作成して、テスト用とアプリケーション用で使い分けられるようにしました。
この記事で話題にしたことは私自身半年ぐらい課題に感じていたことだったので、ある程度解消の目途がたったのは嬉しく思います。
これからはこの機能を用いて、バリバリ機能の実装とテスト実装を行っていきます。
ここまで読んでいただきありがとうございました。

## 参考資料

データベースを複数作成する
[https://ysuzuki19.github.io/post/docker-mysql-postgres-multiple-databases](https://ysuzuki19.github.io/post/docker-mysql-postgres-multiple-databases)
[https://hub.docker.com/\_/postgres#:~:text=and POSTGRES_DB.-,Initialization scripts,-If you would](https://hub.docker.com/_/postgres#:~:text=and%20POSTGRES_DB.-,Initialization%20scripts,-If%20you%20would)
Prisma で複数のデータベースと接続する
[https://px-wing.hatenablog.com/entry/2022/03/04/074543](https://px-wing.hatenablog.com/entry/2022/03/04/074543)
Prisma Import を使用した schema ファイルの結合
[https://webengineer.blog/blogs/hw7xjk0t2i/](https://webengineer.blog/blogs/hw7xjk0t2i/)
[https://zenn.dev/pale_delphinium/articles/96397db45866b0](https://zenn.dev/pale_delphinium/articles/96397db45866b0)
テストデータを削除する
[https://zenn.dev/seya/articles/0aea06f9eb6815](https://zenn.dev/seya/articles/0aea06f9eb6815)
