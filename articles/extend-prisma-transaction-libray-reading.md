---
title: "Prismaのトランザクションを拡張するライブラリのソースコードを読む"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Prisma","NestJs","Promise"]
published: true
---
## この記事の前に
今回の話は非同期処理が結構関わってきます。
ただ、非同期処理については例えば以下のような資料がありますので、この記事では詳細には触れません。
[https://jsprimer.net/basic/async/](https://jsprimer.net/basic/async/)
https://zenn.dev/estra/books/js-async-promise-chain-event-loop
Prismaの設定方法や、Prismaのトランザクション機能についても、詳細は触れません。
また、フレームワークにNestJSを使用しております。
最後に、この記事は自身の理解をまとめる意味合い強いです。
そのため、記載内容には誤りや説明不足があるかと思います。
その際はご指摘いただけますと幸いです。
## はじめに
TypescriptでORMを使用する際、多くの方はPrismaを選ぶと思います。
そのPrismaですが、[$transaction API](https://www.prisma.io/docs/orm/prisma-client/queries/transactions)という一連のDB操作に対してトランザクションを張る機能があります。
これ自体は便利なのですが、例えば[こちらのイシュー](https://github.com/prisma/prisma/discussions/22493)に記載されている以下のコードがあったとします。
```tsx
await prisma.$transaction(async (txn) => {
    await stuff1(txn)
    await stuff2(txn)
  })
```
この時、stuff1やstuff2関数の中でもさらにprismaを使った操作をしていくとなると、トランザクション用のクライアントを渡す階層が深くなってしまいます。
そうなると、各処理がトランザクションに依存することとなり使いにくくなります。
今回は、上記問題を解消するためのライブラリである[@transactional/prisma](https://github.com/Yamanlk/transactional-prisma)の内部実装を確認し、どういったアプローチがあるのかを見ていきます。
結論は以下の通りです。
1. Prismaの$extendsを使用して、一つのトランザクションクライアントを使い回す
2. トランザクションの戻り値をPending状態のPromiseにし、明示的に$commitもしくは$rollbackを実行しないとトランザクションが終わらないようにした。

## @transactional/prismaの実装を見ていく
### ファイルの構成
@transactional/prismaは大きく分けて以下の3ファイルで構成されています。
- [flat-extension.ts](https://github.com/Yamanlk/transactional-prisma/blob/main/src/flat-transaction.ts)
- [extension.ts](https://github.com/Yamanlk/transactional-prisma/blob/main/src/extension.ts)
- [transactional.ts](https://github.com/Yamanlk/transactional-core/blob/main/src/transactional.ts) (別のリポジトリのファイル)

flat-extension.tsはPrismaのトランザクションクライアントをProxyオブジェクトで拡張したものになっています。
そして、extension.tsはflat-extension.tsで作成したトランザクションクライアントを、$extendsで使用するPrismaClientに適用するためのファイルです。
transactional.tsはメソッドデコレーターを作成しています。
### 拡張したトランザクションクライアントを作成するflat-extension.ts
それでは、まず今回の根幹となるflat-extension.tsについて見ていきます。
flat-extension.tsで全ての根幹をになっているのが以下の部分です。
```tsx
  const txPromise = new Promise<undefined>((resolve, reject) => {
    commit = () => resolve(undefined);
    rollback = () => reject(ROLLBACK);
  });
  const txClient: Promise<Prisma.TransactionClient> =
    new Promise<Prisma.TransactionClient>((resolve, reject) => {
      setTxClient = (tx) => resolve(tx);
    });
  const tx: Promise<undefined> = prisma["$transaction"](
    (tx: Prisma.TransactionClient) => {
      setTxClient(tx);
      return txPromise;
    },
    options
  );
```
async-awaitを使っている身としては、Promiseオブジェクトばっかりでドキッとしますが、やっていることは以下の通りです。
- txPromise :commit関数と、rollback関数の実体を定義
- txClient：トランザクションクライアントを取得するsetTxClient関数の実体を定義
- tx：Prismaのトランザクションクライアントを用いて変数txClientのPromiseをFullFilledにし、トランザクションの戻り値をPending状態の変数txPromiseを返す

ここで、重要なのでは変数txの処理です。
変数txでは、変数txClientのPromise内で定義したsetTxClientを実行しています。
setTxClientはPromiseをFullFilledにするresolveを呼び出すようになっているため、変数txを定義した段階で変数txClientで定義したPromiseがFullFilledになります。
そのため、変数txClientをawaitすればPrismaのトランザクションクライアントが取得できます。
実際、後続の処理では以下のように[Proxyオブジェクト内でawait](https://github.com/Yamanlk/transactional-prisma/blob/main/src/flat-transaction.ts#L40C3-L40C34)をし、トランザクションクライアントを取得しています。
```tsx
return new Proxy(await txClient,...)
```
また、変数txはコールバック関数の戻り値を変数txPromiseにしています。
変数txPromiseはこの段階では、まだPendingなPromiseです。
PendingなPromiseを返すということは、変数txPromiseの状態がFulfilledまたはRejectedどちらかの状態にならない限り、トランザクションの処理が完了しないことを示します。
すなわち、変数txPromiseの状態を確定しない限りは、当該トランザクションクライアントを様々な箇所で使用しても、$transaction APIの中での操作と見なすことができそうです。
この部分こそが、「はじめに」で言及したネストしている一連の処理を、トランザクションクライアントを渡さずとも、一つのトランザクション内とみなせる機能を達成するコードになります。
なので、変数txのやっていることさえ掴めれば、8割くらいはライブラリのしたいことをつかめると思います。
最後に、トランザクションクライアントを拡張するためにProxyオブジェクトを返しています。
```tsx
return new Proxy(await txClient, {
        get(target, prop) {
            if (prop === "$commit") {
                return async () => {
                    commit();
                    await tx;
                };
            }
            if (prop === "$rollback") {
                return async () => {
                    rollback();
                    try {
                        await tx;
                    } catch (error: any) {
                        if (error[Symbol.for("prisma.client.extension.rollback")]) {
                            return undefined;
                        }
                        throw error;
                    }
                };
            }
            return target[prop as keyof typeof target];
        },
    }) as FlatTransactionClient;
```
このProxyオブジェクトは、元のオブジェクトに存在していないプロパティの定義も足すことができます。
`if (prop === "$commit")`や`if (prop === "$rollback")`やその部分で、`obj.$commit()`や`obj.$rollback()`と呼び出された時の処理を定義しています。
ちなみに今回は関数をreturnしているので、呼び出す時もメソッドとしていますが、関数を返さなければプロパティとしてアクセスもできます。
詳細については、[ドキュメント](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Proxy)などを参照いただけますと幸いです。
そして、各returnしている関数がやっていることは主に以下の通りです。
- 変数txPromiseの状態を確定している
- すなわち変数txのトランザクションについても成功か失敗したか確定したので、awaitをしてトランザクションを評価している

成功だろうと、失敗だろうとトランザクションを終わらせるための処理を行っています。
以上が、flat-extension.tsの詳細です。
この部分が本来したいことの根幹となるファイルなので、是非とも全体のコードも一度目を通してもらえるとより理解が深まると思います。
### 拡張したトランザクションクライアントをPrismaClientに適用するextension.ts
先ほどまではトランザクションクライアントをどう拡張するかについて見ていきました。
そして、ここでは拡張したトランザクションクライアントをどうPrismaに適用するのかを見ていきます。
Prismaを拡張する部分については、以下[ドキュメント](https://www.prisma.io/docs/orm/prisma-client/client-extensions/query)参照ください。
ざっくり言うと、Prismaで操作する時は必ず設定した処理を経由するようにしています。
拡張方法についてこれくらいに留め、どのように拡張しているかについてここでは見ていきます。
該当部分は以下のコードです。
```tsx
const store = TRANSACTIONAL_CONTEXT.getStore();
/** ...略 */
if (!store.tx) {
  store.tx = new Promise(async (resolve, reject) => {
    const tx = await transaction(prisma, store.options);
    store.$commit = tx.$commit;
    store.$rollback = tx.$rollback;
    resolve(tx);
  });
}
const tx = await store.tx;
```
まず、前提としてTRANSACTIONAL_CONTEXTは後ほど紹介する[transactional.ts](https://github.com/Yamanlk/transactional-core/blob/main/src/transactional.ts)で以下のように定義されているAsyncLocalStorageとなります。
正確な説明は[ドキュメント](https://nodejs.org/api/async_context.html#async_context_class_asynclocalstorage)を見ていただきたいのですが、AsyncLocalStorageについてここでは1リクエストごとに独立して値を共有できるクラスと理解してください。(正確ではないと思うのでご注意ください)
AsyncLocalStorageのgetStoreメソッドは、値が保持されていればそれを取得できます。
そして、取得した値にtxプロパティが存在しなければ、flat-extension.tsで定義したtransaction関数を呼び出しています。
これによって、prismaから拡張したトランザクションクライアントを取得できたので、AsyncLocalStorageのプロパティにセットしています。
そして、Prisma自体が対象のトランザクションクライアントを用いてDB操作をしないと意味がないので、storeにセットしたトランザクションクライアントを`const tx = await store.tx;`で格納しています。
後続の処理で、そのクライアントを使用し、Prismaの処理を行っています。
以上をまとめると、extension.tsは以下のことを行っています。
- AsyncLocalStorageに拡張したトランザクションクライアントを保存し、共有している
- 通常のPrismaによる操作も、拡張したトランザクションクライアントを使用するようにした

ちなみに、extension.tsで定義したものをNestJSで適用しようとすると型エラーが起きます。
なので、拡張する場合は以下のように実装します。
```tsx
import { Injectable, Logger, OnModuleInit } from '@nestjs/common';
import { Prisma, PrismaClient } from '@/__generated__';
import { prismaTransactional } from '@/transaction/extension';
const extendClient = (client: PrismaClient<Prisma.PrismaClientOptions, Prisma.LogLevel>) => {
    return client.$extends(prismaTransactional)
}
class UntypedExtendedPrismaClient extends PrismaClient<Prisma.PrismaClientOptions, Prisma.LogLevel> {
    constructor() {
        super({
            log: [
                'error',
                'warn',
                'info',
                'query',
            ]
        })
        return extendClient(this) as this
    }
}
const ExtendedPrismaClient =
    UntypedExtendedPrismaClient as unknown as new () => ReturnType<
        typeof extendClient
    >
@Injectable()
export class PrismaService extends ExtendedPrismaClient implements OnModuleInit {
    async onModuleInit() {
        await this.$connect();
    }
}
```
これは以下のQiitaの記事を元に実装していますので、詳しい話は記事を参照ください。
https://qiita.com/irohafox/items/2b4b7ce87963a25f4f63
以上の拡張を行うことで、Prismaを使った操作は、拡張したトランザクションクライアントを使う形で操作できるようになりました。
### 今までの内容を適用するデコレーター作成transactional.ts
ここまでで、Prisma関連の拡張が終わったので、後はトランザクションを解放するメソッドを呼び出すようにします。
今回紹介しているライブラリでは、それを@Transactional というメソッドデコレーターを使うことで、達成しています。
デコレーターの実装は以下の通りです。
```tsx
export const Transactional =
  <O = any>(propagation: Propagation = Propagation.REQUIRED, options?: O) =>
  (target: any, propertyKey: string, descriptor: PropertyDescriptor) => {
    const originalMethod = descriptor.value;
    if (descriptor.value) {
      descriptor.value = transactional(originalMethod, propagation, options);
    }
  };
```
色々書いていますが、元のメソッドをtransactionalという関数の戻り値で上書きしているだけです。
そのため、[transactional.ts](https://github.com/Yamanlk/transactional-core/blob/main/src/transactional.ts)内にあるtransactional関数のコードを一部みていきます。
主要な部分の実装は以下の通りです。
```tsx
export const transactional = <
  O extends any,
  T extends (...args: any) => Promise<any> = any
>(
  method: T,
  propagation: Propagation = Propagation.REQUIRED,
  options?: O
): T => {
  return async function (this: any, ...args: any[]) {
    const store = TRANSACTIONAL_CONTEXT.getStore();
    if (
      [Propagation.REQUIRED, Propagation.MANDATORY].includes(propagation) &&
      store
    ) {
      return method.call(this, ...args);
    }
    if (Propagation.REQUIRED === propagation && !store) {
      return TRANSACTIONAL_CONTEXT.run({ options }, async () => run.call(this));
    }
    
    /** ...略 */
    async function run(this: any) {
      try {
        const result = await method.call(this, ...args);
        const store = TRANSACTIONAL_CONTEXT.getStore();
        if (store?.$commit) {
          await store.$commit();
        }
        return result;
      } catch (error) {
        const store = TRANSACTIONAL_CONTEXT.getStore();
        if (store?.$rollback) {
          await store.$rollback();
        }
        throw error;
      }
    }
  } as T;
};
```
AsyncLocalStorageに値が存在すれば、元のメソッドをそのまま実行するようにし、なければrun関数内で元のメソッドを実行します。
元のメソッドが成功すれば、[flat-transaction.ts](https://github.com/Yamanlk/transactional-prisma/blob/main/src/flat-transaction.ts)で設定した$commitを呼び出し、失敗したら$rollbackを呼び出します。
$commitや$rollbackはトランザクションの解放を行うので、一連の処理内でのトランザクションを終了させることができます。
ちなみに、storeがある場合のケースは@Transactionalデコレーターを付与しているメソッドの中で、@Transactionalデコレーターを付与しているメソッドを呼び出す場合です。
内部のメソッドでトランザクションを解放しないように、元のメソッドをそのまま呼び出しています。
以上が主要なコードについての解説です。
なかなか直ぐに腑に落ちないかもしれないですが（実際自分はこの週末を犠牲にして少しだけ納得できたかな？くらいの理解度です）、使っていくうちに慣れていくと思うので是非試してみてください。
## おわりに
今回は、https://github.com/Yamanlk/transactional-prismaの内部実装をみていきました。
それによって、Prismaの$transaction API内のクライアントを様々なメソッドで渡さずに、メソッド全体の処理にトランザクションを張ることを知ることができました。
それに伴い、非同期処理の理解が全然甘かったのも痛感しました。
全部を理解しきれてはいないので、今後非同期処理についても理解を深めて行けたらと思います。
ここまで読んでいただきありがとうございました。