---
title: "インターセプターでイベントを発行する(NestJS)"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NestJS", "EventEmitter"]
published: true
---

## はじめに

NestJS を使った API で、通知処理などを行うためにイベント発行の処理を入れていました。
その処理は主に、Controller クラス内に記載しており、都度呼び出すようにしていました。
しかし、このやり方では様々なエンドポイントに対して、都度イベント発行処理を記載する必要があります。
このやり方でもできないことはないですが、実装漏れがおきやすくなると感じていました。
さらに、このイベント発行は基本的に全体の処理が終わった後に行うつもりでいます。
そこで、いっそのことイベント発行をインターセプター内で行い、レスポンスのタイミングで行うようにしました。
それによって、コントローラー内に都度イベント発行の処理を書かずに済みます。
さらに、インターセプターを一工夫することで、イベント発行するエンドポイントと、そうではないエンドポイントを漏れが少ない状態で、定義できるようにもなります。
今回はその実装について見ていきます。

## 全体的なコード

### インターセプター部分

```jsx
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { tap } from 'rxjs/operators';
import { LogCreatedEvent } from './log-created.event';
import { AppController } from 'src/app.controller';
const methodsConfig: { [key in keyof AppController]: boolean } = {
    getProfile: false,
    login: true,
    testMethod: true
}
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
    constructor(private readonly eventEmitter: EventEmitter2) { }
    intercept(context: ExecutionContext, next: CallHandler) {
        const methodName = context.getHandler().name
        return next
            .handle()
            .pipe(
                tap(() => {
                    if (methodsConfig[methodName]) {
                        this.eventEmitter.emit('log.created', new LogCreatedEvent({ id: 'id', name: 'name' }))
                    }
                })
            );
    }
}
```

### イベントクラス

```jsx
type LogEventData = { id: string, name: string }
export class LogCreatedEvent {
    constructor(readonly eventData: LogEventData) { }
}
```

### コントローラークラス（一部省略しています）

```jsx
import { Controller, UseInterceptors } from "@nestjs/common";
import { LoggingInterceptor } from "./interceptor/logging.interceptor";
@UseInterceptors(LoggingInterceptor)
@Controller()
export class AppController {
  testMethod() {
    return "test";
  }
  async login() {
    /** ログイン処理 */
  }
  getProfile() {
    return "profile";
  }
}
```

今回の内容に関わるコードは上記の通りですが、このコードをそのまま使用してもイベントは発行されません。
実際は、イベントハンドラークラスを登録することや NestJS の Module にイベントモジュールを設定する必要があります。
ですが、その点については[ドキュメント](https://docs.nestjs.com/techniques/events)に記載してありますので、ここでは省略いたします。
コード全体を示したのですが、コードそのものを解説する前に、まずはインターセプターとイベントについて簡単に見ていきます。

## NestJS におけるインターセプター

### インターセプターとは

NestJS にはインターセプターという機能があります。
機能の中身としては、以下画像のようにリクエストとレスポンスの時に、任意の処理を差し込むものになります。
![[https://docs.nestjs.com/interceptors](https://docs.nestjs.com/interceptors) より引用](/images/interceptor-event-in-nestjs/image.png)
[https://docs.nestjs.com/interceptors](https://docs.nestjs.com/interceptors) より引用
このインターセプターですが、 [Aspect Oriented Programming](https://en.wikipedia.org/wiki/Aspect-oriented_programming) (AOP)をもとに構築されています。
そのため、以下のことが可能となっています。

- メソッドを実行する前と後に追加でロジックを挿入できる
- Route Handler から返ってくる値を変換できる
- Route Handler からの例外処理を変換できる
- 基本的な関数のふるまいを拡張できる
- 特定の条件（キャッシュの有無など）に依存した Route Handler を上書きできる

では実際にインターセプターを実装してみます。

### インターセプターの簡単な実装

ここでは[ドキュメント](https://docs.nestjs.com/interceptors)の内容をもとに、インターセプターを実装していきます。
まずは、インターセプター用のクラスを作成します。

```jsx
import { Injectable, NestInterceptor } from "@nestjs/common";
@Injectable()
export class LoggingInterceptor implements NestInterceptor {}
```

もちろんこれだけでは動きませんが、インターセプタークラス自体はまず以下の定義をします。

1. @Injectable デコレーターを付与し、注入可能なクラスにする
2. NestInterceptor インターフェースを継承する

この二つを行うことで、インターセプタークラスであることを確定させます。
次に内部のメソッドを実装します。
先程継承した NestInterceptor インターフェースは以下定義となっています。

```jsx
export interface NestInterceptor<T = any, R = any> {
  intercept(
    context: ExecutionContext,
    next: CallHandler<T>
  ): Observable<R> | Promise<Observable<R>>;
}
```

そのため、intercept メソッドを実装します。
この intercept メソッドはそれぞれ二つの引数を受け取り、それぞれの機能はざっくり以下の通りです。

- リクエスト情報などが含まれている[ExecutionContext](https://docs.nestjs.com/fundamentals/execution-context)
- handle メソッドを持ち、RxJS の[Observable](https://rxjs.dev/guide/observable)を返す CallHandler

戻り値については、Promise か Observable を返す必要がありますが、個人的には Observable を返す方が扱いやすいと感じています。
というより、Promise を返すときの資料が見当たらなくて、私はどうしてよいかわかりませんでした。
そのため、この先は Observable を返す前提で話を進めます。
Observable を返す場合、CallHandler の handle メソッドを実行する必要があります。
handle メソッドは Observable を返しますし、CallHandler はレスポンスの情報を持っています。
なので、CallHandler の handle メソッドを呼ぶことで、レスポンスの情報をもった Observable を返すことができます。
以上から、intercept メソッドの最小実装は以下の通りです。

```jsx
intercept(context: ExecutionContext, next: CallHandler) {
	return next
          .handle()
}
```

この実装はレスポンスをそのまま返しているだけなので、何もしていません。
上記実装だけでは、インターセプターを作成した意味はないので、実際にはここに様々な処理を足していくことになります。
処理を足していくために、どこがどのタイミングで動くかも見ていきます。
基本的に、CallHandler の handle メソッドが実行されるまでは、Route Handler に到達する前、すなわちリクエスト部分で実行されます。
よって、インターセプターでリクエストを加工したり、リクエストデータを取得したい場合は、CallHandler の handle メソッド実行より前に記載します。
一方、レスポンス情報を加工したいときは、Observable が持つ[pipe メソッド](https://rxjs.dev/api/index/function/pipe)を用います。
この pipe メソッド内で RxJS が提供する様々な関数を設定することで、レスポンスの際にデータの加工であったり、副作用を伴う処理を実行できます。
RxJS 提供の関数は様々ものがありますが、レスポンスのデータを変換したい場合は、以下のように map 関数を使用します。

```jsx
intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
 }
```

この場合、map 関数はレスポンスデータを受け取るので、適宜値を加工できます。
また、レスポンスの時に、変換ではなく副作用な処理を実行したい場合は、以下のように pipe メソッド内で tap 関数を使用します。

```jsx
intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(
        tap(() => console.log(`レスポンスの時に出力させる`)),
      );
  }
```

以上簡単な説明ですが、今言った内容は[ドキュメント](https://docs.nestjs.com/interceptors)にも記載があるので、詳細はそちらを確認してください。

## NestJS でのイベント発行

Node.js には[Observer パターン](https://ja.wikipedia.org/wiki/Observer_%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3)を実装した、[EventEmitter](https://nodejs.org/en/learn/asynchronous-work/the-nodejs-event-emitter)というものがあります。
Node.js ですでに存在していることから、NestJS でも上記機能を使用することができます。
ただ、Node.js と同じ内容を実装する必要はなく、[EventEmitter2](https://github.com/EventEmitter2/EventEmitter2)というパッケージを内部使用している、@nestjs/event-emitter パッケージを使い実装します。
これによって、一定のお作法で簡単にイベント処理を実装できます。
実装方法は大きく分けて以下のようになります。

1. イベントモジュールを AppModule に登録
2. イベントを受け取り、処理するハンドラークラスを作成する
3. ハンドラークラスが受け取るデータを持つ、クラスを作成する
4. https://github.com/EventEmitter2/EventEmitter2を使い、3のクラスと2を起動するためのイベント名を設定した状態で、イベント発行を行う

コードの書き方についてもこの記事で見ていこうと思いましたが、ほぼ[ドキュメント](https://docs.nestjs.com/techniques/events)に書いてあるので、詳細についてここでは記載しません。
ドキュメントと[サンプルコード](https://github.com/nestjs/nest/tree/master/sample/30-event-emitter)を見ると、大体雰囲気はつかめると思います。

## コードの解説

主要な登場人物を紹介したので、改めて今回のコードで重要なインターセプター部分の処理の解説を記載します。

```jsx
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { tap } from 'rxjs/operators';
import { LogCreatedEvent } from './log-created.event';
import { AppController } from 'src/app.controller';
const methodsConfig: { [key in keyof AppController]: boolean } = {
    getProfile: false,
    login: true,
    testMethod: true
}
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
    constructor(private readonly eventEmitter: EventEmitter2) { }
    intercept(context: ExecutionContext, next: CallHandler) {
        const methodName = context.getHandler().name
        return

    }
}
```

主に以下のことを行っています。

- クラスのメソッドをプロパティに持つオブジェクトを作成
- Route Handler の名前を取得
- レスポンス内でイベントを発行

それぞれ見ていきます。

### クラスのメソッドをプロパティに持つオブジェクトを作成

この実装は以下部分です。

```jsx
const methodsConfig: { [key in keyof AppController]: boolean } = {
    getProfile: false,
    login: true,
    testMethod: true
}
```

意図としては、Controller クラスでイベントを発行するメソッドか否かを定義するためです。
Controller クラスが持つメソッドをプロパティとし、その値が true の時だけイベントを発行したいと思っています。
さらに、各エンドポイントに対して、設定漏れを無くしたいとも思っています。
それらを達成するために、`{ [key in keyof AppController]: boolean }`で型定義をすることで、対象クラスのメソッド全てをプロパティに定義することを強制しています。
これによって、各種エンドポイントがイベントを発行するか否かの設定を強制することができ、要件を満たせます。

### Route Handler の名前を取得

先程エンドポイントのメソッド名をプロパティに持つ、オブジェクトを作成しました。
そのオブジェクトは、イベントの発行を行うか否かの設定値となることも確認しました。
であれば、インターセプター内でどのメソッドが実行されたかの情報を取得する必要があります。
上記要件は以下で達成しています。

```jsx
const methodName = context.getHandler().name;
```

ExecutionContext はこれから実行する Route Handler の情報を持っており、それは getHandler メソッドで取得できます。
この、実行する Route Handler とは Controller クラスのメソッドとなります。
であれば、メソッドが持つ name プロパティにアクセスすれば、メソッド名を取得することができます。
イメージが湧いていない方は、[こちらのプレイグラウンド](https://www.typescriptlang.org/play/?#code/MYewdgzgLgBAFiA5gUxgXhgCgJRoHwDeokIANsgHSlKYDkUy0t2AvgFBvERmXWKYIUFMAEMAtsmxA)を確認してください。
関数を実行するのではなく、name プロパティにアクセスすることで、関数名だけが取得できると思います。
今回行っている処理も同じことです。
以上で、実行するコントローラーのメソッド名が取得できました。
なお、このメソッド名を取得する処理ですが、REST でも gRPC どちらでも同様にメソッド名を取得できることを確認しています。
後は、これを先程作成したオブジェクトのアクセスキーとして使うことで、イベントを発行するフラグが有効かどうかを判断できます。
最後にその部分の処理をみていきます。

### レスポンス内でイベントを発行

レスポンスデータは、CallHadller の handle メソッドの戻り値である Observable です。
なので、この Observable に設定を追加することでイベント発行処理を差し込むようにします。
具体的には以下の通りです。

```jsx
next.handle().pipe(
  tap(() => {
    if (methodsConfig[methodName]) {
      this.eventEmitter.emit(
        "log.created",
        new LogCreatedEvent({ id: "id", name: "name" })
      );
    }
  })
);
```

イベント発行は副作用を伴う処理なので、tap 関数内で実行するようにしています。
`if (methodsConfig[methodName])`とすることで、イベント発行を実行するエンドポイントかを振り分けています。
この値が true であれば、イベント発行を想定したエンドポイントなので、EventEmitter2 でイベントを発行しています。
後は、事前に定義したイベントハンドラーが起動し、対象の処理が実行されます。
これでインターセプターは完了したので、イベント発行するかを設定するオブジェクトを作成するときに使用したコントローラークラスに、`@UseInterceptors`デコレーターを付与し、引数にインターセプタークラスを指定します。

```jsx
import { Controller, UseInterceptors } from "@nestjs/common";
@UseInterceptors(LoggingInterceptor)
@Controller()
export class AppController {
  /** 省略 */
}
```

以上で、イベント処理を独立させつつ、設定もれにくい仕組みが完成しました。
イベント部分は独立させて実装したいと思っている方の参考になれば幸いです。

## 今回の実装の懸念点

これまでの実装で、イベント発行を一つのインターセプターへ集約することができました。
これ自体は個人的に便利だと思っている反面、懸念点が 3 つあります。
一つ目は、メソッド名が被るとイベント発行の設定が上手く効かないことです。
例えば、以下のコードをみてください。

```jsx
const test = {
  execute: true,
}
const test2 = {
  execute: false,
}
const config: {
  [key in keyof typeof test]: boolean }
  & { [key in keyof typeof test2]: boolean
}
```

test と test2 はそれぞれ同じプロパティをもっています。
各プロパティを抽出した型を&で繋いだ変数 config を定義すると、以下のように一つのプロパティしか定義できません。
![2024-09-01_18h15_13.png](/images/interceptor-event-in-nestjs/2024-09-01_18h15_13.png)
このことから、同じメソッド名が存在すると、片方のメソッドはイベント発行を行うが、もう片方はイベント発行をしないという設定ができません。
上記を避けるために、各 Controller クラスに対してインターセプターを作成するか、メソッド名は必ずプレフィックスをつけるなどの設定が必要です。
また、インターセプターにまとめてしまうことは、自動で型推論をしてくれる Typescript の旨味を消すことにもなります。
これは、インターセプターがリクエストやレスポンスのデータを全て any 型として持っているためです。
そのため、今回のケースでいうと、レスポンスの値を使用したいとなったら、以下のように明示的に返ってくる型を定義しなければなりません。

```jsx
next.handle().pipe(
  tap <
    { responseId: string } >
    ((res) => {
      /** 省略 */
    })
);
```

これで動くとはいえ、エンドポイントの戻り値を都度確認しないといけないので、引数としてレスポンスデータを使う場合は、注意が必要になります。
最後に、インターセプターの特性として、リクエスト・レスポンスデータしか扱うことができません。
そのため、処理の中間の値を使用して、イベント発行をしたいと思ってもできません。
その場合は、コントローラークラスやサービスクラスにイベント発行処理を実装する必要があります。
あくまで、イベント発行をインターセプターで行うのは、限られたケースでしか使うことはできないです。
以上のように、インターセプターでイベントを発行することはメリット・デメリットがあるので、イベントの種類や、要件などによって使用するかは判断する必要があります。

## おわりに

今回はインターセプターを用いて、イベントの発行を切り出すようにしました。
イベントの発行を一つにまとめることができたことで、管理はしやすくなったように思います。
もちろん課題は存在するので、いかなるケースで使用することはできませんが、一つの選択肢を見出せたのは良かったです。
また、インターセプターについての理解も深めることができたので、試してみて良かったなと思います。
今回紹介した処理が使えそうであれば、是非とも使ってもらえると幸いです
ここまで読んでいただきありがとうございました。
