---
title: "テスト実行時にPrismaが参照するデータベースのURLを変更する"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["prisma", "jest"]
published: true
---

## はじめに

[以前の記事](https://zenn.dev/maronn/articles/prisma-databases-for-test)で、Prisma のスキーマファイルをテスト用とアプリ用で作成し、PrismaClient をモックせずにテストを行うための処理を実装しました。
しかし、実際に使用してみると複数の問題が出てきました。
まずは、運用になれるまで誤ったスキーマファイルにテーブルを定義してしまい、適切にデータベースに反映されない事態が発生しました。
次に、なぜかアプリケーション用のデータベース URL が適切な値になっているはずなのに、テスト用のデータベースを読み込んでしまう不具合がプロジェクトによっては発生しました。
こちらは原因が分かっておらず、かなり致命的な問題となっています。
このように、スキーマファイルを複数に分割するのはまだまだ課題を抱えていました。
そこで、今回は Jest でテスト実行時にデータベースの URL を直接書き換えることで、一つのスキーマファイルでテスト用とアプリケーション用の PrismaService を切り替えられるようにしました。
前回よりも簡単に設定ができるのと、一度設定してしまえば今後そのファイルについて触ることはあまりないので、設定漏れや記載場所の運用を考える必要がありません。
なお、今回は[NestJS](https://nestjs.com/)環境下で話をしています。
特に NestJS でないと適用できないわけではありませんが、前提として把握お願いします。
また、Ubuntu 環境下のみで動作確認を行っています。
そのため、別の環境下で動くことを保証できないことはご留意ください。

## この記事のここだけ！！！

[dotenv-cli](https://www.npmjs.com/package/dotenv-cli)をインストールします。
プロジェクトの package.json の scripts に以下の処理を記載します。

```tsx
"test": "expot DATABASE_URL=$(dotenv -- bash -c 'echo $DATABASE_TEST_URL')  && prisma db push && jest "
```

これを行えばテスト用のデータベースに接続できます。

## 環境構築

本丸の話題の前に、まずテストが実行できる準備をします。

### データベースの用意

こちらの記事をもとに docker-compose.yaml を使って、テスト用とアプリケーション用のデータベースを作成してください。

### PrismaService とスキーマファイルの用意

[NestJS のクイックスタート](https://docs.nestjs.com/first-steps#setup)を基にプロジェクトを作成したら、.env ファイルに以下の環境変数を定義します。

```tsx
DATABASE_URL = "postgresql://postgres:postgres@db:5432/postgres?schema=public";
```

そして、Prisma のクイックスタートの「[Model your data in the Prisma schema](https://www.prisma.io/docs/getting-started/quickstart#2-model-your-data-in-the-prisma-schema)」まで完了させ、`npx prisma db push`でデータベースの定義を反映させてください。
その後、npx prisma studio で表示された Prisma Studio で user テーブルにアクセスし、画面上部の「Add Record」をクリックして、任意のデータを登録します。
これでアプリケーション用のデータベースにデータが保存されました。
完了したら、プロジェクトの src 配下の任意の場所に以下の prisma.service.ts と prisma.module.ts を作成します。
prisma.service.ts

```tsx
import { Injectable } from "@nestjs/common";
import { PrismaClient } from "@prisma/client";
@Injectable()
export class PrismaService extends PrismaClient {}
```

prisma.module.ts

```tsx
import { Module } from "@nestjs/common";
import { PrismaService } from "./prisma.service";
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

これで Prisma 側の準備は完了です。
以前よりもやることが大分減りましたね。

### テスト環境の用意

実は NestJS はプロジェクトを構築した時点で、Jest の設定はある程度行っております。
そのため、以下のようなテストファイルを作成するだけでテストの準備は完了しています。

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

テストの構文自体は[ドキュメント](https://docs.nestjs.com/fundamentals/testing)に詳細を委ねますが、簡単に説明すると Test.createTestingModule で必要なコントローラーやサービスを呼び出します。
そして、テストしたいメソッドが存在するクラスを get メソッドで取得します。
後は it や test などのテストケースを示す関数の中で、実際にメソッドを呼び出し期待する結果を expect を使い、比較します。
ちなみに、getUsers メソッドは以下の処理が記載されており、このテストはアプリ用のデータベースを起動したらテストは通るようになっています。

```tsx
async getUsers() {
    return await this.prismaService.user.findMany()
  }
```

ここまでで、テストを行う準備はできたのでテスト実行する時のみ、テスト用のデータベースと接続している PrismaService を使用するようにします。

## テスト実行時に使用するデータベースを切り替える

PrismaService は環境変数で設定した DATABASE_URL の値をもとに、接続するデータベースを特定しています。
となると、PrismaService を実行する際に DATABASE_URL を書き換えることができれば目的を達成できそうです。
なので、ここでは上記の目的を果たすための処理を行っていきます。
ただ、やることは単純で package.json のテストを実行する script を以下のように環境変数を書き換えるようにすれば目的は果たせます。

```tsx
"test": "export DATABASE_URL=postgresql://postgres:postgres@db:5432/postgres_test?schema=public  && npm run db:push:local && jest "
```

これで、テストを実行してみると以下のようにテストが失敗します。
![Untitled](/images/ovverite-env-in-exec-jest/Untitled.png)
やりましたね。
試しに、先頭の環境変数を書き換える処理をなくせば、今度はテストが通ることを確認できます。
テスト用のデータベースには何もデータを投入しておらず、アプリケーション用のデータベースにはデータが存在しているので参照するデータベースが振り分けられていそうです。
ちなみにですが、export で環境変数を書き換えていますが、これはコマンドが完了するまでの間のみ有効です。
そのため、NestJS を実行する任意の TS ファイルに`console.log(process.env.DATABASE_URL)`を記載すると、アプリケーション用のデータベース URL が表示されると思います。
これで目的の動作は達成できました。
ただ、テスト用のデータベースとはいえデータベースの URL をそのままコマンドに記載するのは不安です。
また、テスト用のデータベース URL が変更になった際都度書き換える必要があります。
それは手間なので、最後に環境変数から値を取得し、その値をもとに DATABASE_URL を書き換えるようにします。
まずは.env ファイルに以下のようなテスト用のデータベースの URL が記載されている値を追記します。

```tsx
DATABASE_TEST_URL =
  "postgresql://postgres:postgres@db:5432/postgres_test?schema=public";
```

次にコマンドで環境変数の値を取得するために、[dotenv-cli](https://www.npmjs.com/package/dotenv-cli)をインストールします。
ターミナルなどで`npm i -D dotenv-cli`を実行します。
インストールができたら、環境変数を書き換えている部分を以下のように変更します。

```tsx
export DATABASE_URL=$(dotenv -- bash -c 'echo $DATABASE_TEST_URL')
```

環境変数 DATABASE_URL は「$( )」で囲まれたコマンドの結果を格納するようにします。
なので、「$( )」の中身である`dotenv -- bash -c 'echo $DATABASE_TEST_URL'`について見ていきます。
まず、dotenv コマンド部分は以下の構文です。

```tsx
dotenv (オプション)  -- 実行したいコマンドを指定
```

「--」を挟んだ左側では、環境変数を読み込んだりするためのオプションを設定します。
特にオプションを設定しなかった場合は、.env ファイルの値を読み込みます。
そして、右側では実行するコマンドを指定します。
実行するコマンドは任意のコマンドで構いません。
そのため、今回は環境変数`DATABASE_TEST_URL`を出力させるコマンドを定義しています。
これによって、まず dotenv コマンドで.env ファイルの値を取得し、その後環境変数`DATABASE_TEST_URL`を出力させることができます。
出力した結果を export コマンドを使い、環境変数 DATABASE_URL に格納するようにしています。
以上で、コマンドにテスト用のデータベース URL を記載せずとも、環境変数から値を取得し、テストの時だけ参照するデータベースを切り替えることができるようになりました。
実際に`npm run test`でテストを実行すると、テスト用のデータベースに振り分けられ、テストがおちることを確認できます。

## おわりに

今回は単一のスキーマファイルでテスト用のデータベースとアプリケーション用のデータベースを切り替えられるようにしました。
コマンドで切り替えるのは中々制限が多く、解決策を見出すのが大変でしたが、案外解消方法は単純でした。
また、scripts コマンドで環境変数を書き換えても、そのコマンドの実行中しか反映されないのは不便だと思っていましたが、今回は非常に助かりました。
以前にも比べ、格段に保守がしやすいデータベース定義周りになったので、Prisma を使ってテストはしたいが、モックはあまりしたくない人は是非ともお試しください。
ここまで読んでいただきありがとうございました。
