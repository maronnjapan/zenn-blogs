---
title: "DataLoaderのloadメソッドが何をしているのかを確認する"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GraphQL", "DataLoader", "NodeJs"]
published: true
---

## この記事を読む前の注意点

[こちらの記事](https://zenn.dev/dino3616/articles/c726ec63b11775)にあるような、load メソッドを呼び出すところまではできている人を対象に記載しています。

```tsx
@Resolver(() => Post)
export class PostResolver {
  constructor(private readonly userDataLoader: UserDataLoader) {}
  @ResolveField(() => User)
  async user(@Parent() post: PostModel): Promise<UserModel> {
    const user = await this.userDataLoader.load(post.userId);
    return user;
  }
}
```

また、GraphQL とは何ぞや、N+1 問題とは何かといった、DataLoader を使うためのきっかけになる事象についても記載しません。
それらについても、ふんわりと把握している前提で記載しています。
あくまで、DataLoader は何をしているかを解説し、それによって N+1 問題が解消できるのを示すことのみに焦点を当てていきます。

## はじめに

GraphQL を使う上で避けては通れない問題の一つに、N+1 問題があります。
これを解消するために Google で検索してみると、DataLoader を使った解消方法が多く見つかります。
このことから、N+1 問題を解消するために DataLoader を使うという流れは多くの GraphQL を使用する人が到達すると思います。
そして、DataLoader を行うためのライブラリは多く存在し、Javascript ベースのものも[graphql/dataloader](https://github.com/graphql/dataloader)というライブラリが存在します。
[graphql/dataloader](https://github.com/graphql/dataloader)は以下のようにバッチ処理したい関数を定義し、

```tsx
const DataLoader = require("dataloader");
const userLoader = new DataLoader((keys) => myBatchGetUsers(keys));
```

load メソッドを呼び出すことで都度実行ではなく、load メソッドの引数に設定した値をまとめて使用して、定義したバッチを実行します。※

```tsx
const user = await userLoader.load(1);
```

なので、多少の制約はありますが、DataLoader を行うための手順は以下の二つに集約されます。
① 実行したい関数を作り、その関数を渡した DataLoader クラスをインスタンス化する
②load メソッドで ① の関数を実行する際に必要な ID を渡します。
これで、対象となる ID が全部 load メソッドに渡されるまではクエリ処理が行われず、N+1 問題を解消できます。
ここまで読んで、DataLoader を実装された方であれば、雰囲気は伝わったかなと思います。
ただ、実際にこの DataLoader ライブラリを使うと、以下のような違和感を感じると思います。
① なぜ load メソッドに ID が渡されるまで処理が止まるのか。
② そして、load メソッドはなぜ一つづつ値をかえすのか。
今回はそれら疑問について、ライブラリ内部のコードをみることで解消していこうと思っています
※Github を見てもらえば分かると思いますが、load メソッド以外に似たものとして loadMany があります。なので、厳密には load メソッドと loadMany 二つ使えるのですが、内部の処理は概ね一緒なので、今回は load メソッドのみを話題にしています。

## この記事のここだけ!!!

DataLoader クラスの load メソッドは単純化すると以下の通りになります。

```tsx
class DataLoader {
  _batch = { hasDispatched: false, keys: [], callbacks: [] };
  constructor(batchLoadFn) {
    this._batchLoadFn = batchLoadFn;
  }
  load(key: K): Promise<V> {
    if (!this._batch.hasDispatched) {
      this._batch.hasDispatched = true;
      process.nextTick(() => {
        const batchPromise = loader._batchLoadFn(batch.keys);
        batchPromise.then((values) => {
          // Step through values, resolving or rejecting each Promise in the batch.
          for (let i = 0; i < batch.callbacks.length; i++) {
            const value = values[i];
            if (value instanceof Error) {
              batch.callbacks[i].reject(value);
            } else {
              batch.callbacks[i].resolve(value);
            }
          }
        });
      });
    }
    this._batch.keys.push(key);
    const promise = new Promise((resolve, reject) => {
      this._batch.callbacks.push({ resolve, reject });
    });
    return promise;
  }
}
```

やっていることは以下の二点です。
① 同期処理による、プロパティへの値セット
②process.nextTick で同期処理の後に処理を実行するようにする

## DataLoader クラスの流れを理解するために必要なこと

ここでは DataLoader の処理を理解するために抑えておきたいことについて触れていきます。
ちゃんと理解するにはイベントループについて理解する必要があります。
ただ、ここでは私がちゃんとイベントループを理解しきれておらず、正確な説明ができません。
なので、あくまで内部の挙動は雰囲気程度で捉えてもらい、実行手順が変わるんだとだけ把握してもらえると幸いです。

### Node.js(Javascript)の処理について

![[描いて理解するイベントループ](https://qiita.com/ist-ko-su/items/a985864fc767d8bdfc4a) より引用](/images/dataloader-load/2024-04-14_15h42_37.png)
[描いて理解するイベントループ](https://qiita.com/ist-ko-su/items/a985864fc767d8bdfc4a) より引用
Node.js はシングルスレッドで処理が行われています。(Worker Thread は今回無視します)
なのでコードで実装した動作についてはそれぞれ、順番に実行されます。
基本の流れは上記のとおりですが、今のところ全て同期的な処理の場合の動きでしかありません。
ここに今回大切になってくる、非同期処理が挟まると以下のようになります。
![[描いて理解するイベントループ](https://qiita.com/ist-ko-su/items/a985864fc767d8bdfc4a) を一部改変](/images/dataloader-load/node.png)
[描いて理解するイベントループ](https://qiita.com/ist-ko-su/items/a985864fc767d8bdfc4a) を一部改変
非同期処理の場合、とりあえず非同期処理ですよという宣言だけタスクとして登録します。
ただし、実際の処理は登録したタイミングでは行わず、後で行うようにしています。
このとりあえず処理があるよということだけ登録して、実際の処理は後で行うというのが今回とても大事になってきます。
なお、後で行うとした処理はタスクが一通り片付いてから実行されます。
それを示すために以下のサンプルコードをみてください。

```tsx
const promise = new Promise((resolve) => {
  resolve("非同期処理");
});
promise.then((res) => console.log(res));
for (let i = 0; i < 10; i++) {
  console.log(i);
}
```

Promise を定義し、成功時には「非同期処理」という文言を渡すようにしています。
そして、`promise.then((res)=>console.log(res))`で成功した時の処理を登録しています。
その後に、for 文でループ処理を行っています。
このサンプルコードの実行結果は以下の通りです。

```tsx
0;
1;
2;
3;
4;
5;
6;
7;
8;
9;
非同期処理;
```

今回は同期処理で積まれたタスクは for 文でループ処理を行うことでした。
なので、非同期処理である「非同期処理」を表示させる処理は前段であるループ処理を行った後に、実行されるようになります。
ここまでで Node.js の処理と非同期処理が存在する場合の流れについてざっくり見てきました。
不正確・かつ比喩的な説明になってしまいましたが、このことを頭に入れて今回の処理に関わってくる登場人物を紹介していきます。

### 登場人物 ① process.nextTick

ここでは今回最も大切で、かつ馴染みのない?process.nextTick について紹介します。
まずは process.nextTick は[ドキュメント](https://nodejs.org/api/process.html#processnexttickcallback-args)を確認すると以下説明があります。

> `process.nextTick()` adds `callback` to the "next tick queue". This queue is fully drained after the current operation on the JavaScript stack runs to completion and before the event loop is allowed to continue.

翻訳すると以下のような感じです。

> process.nextTick() は「次のティックキュー」にコールバックを追加します。このキューは、JavaScript スタックでの現在の操作が完了まで実行された後、イベント ループの続行が許可される前に完全に空になります。

よく分かりませんね。
正確に理解していこうとすると、Node.js の Event Loop であったり、Macrotasks や Microtasks の理解であったりが必要だと思います。
ですが、今回は process.nextTick を以下のように扱います。
**process.nextTick は引数に設定した関数を非同期にする。ただし、Promise より前に実行される**
非同期処理は先程見たように、とりあえず処理があることだけ伝えて後で実行するものとなっています。
その非同期処理だと示すのは Promise 以外に process.nextTick も使用できます。
例えば以下のサンプルコードをみてください。

```tsx
console.log("start");
process.nextTick(() => {
  console.log("nextTick callback");
});
console.log("scheduled");
```

ぱっと見非同期処理はどこにも存在しません。
ですが、これを実行すると以下の順番で出力されます。

```tsx
start
scheduled
nextTick callback
```

先程見た Promise と似たような挙動になりますね。
このように process.nextTick は引数の関数を非同期処理にすることができます。
Promise ととても似ていますが、一個明確な違いがあります。
それが Promise が実際に行う処理より前に実行されるという点です。
例えば以下のサンプルコードをみてください

```tsx
new Promise((resolve) => {
  resolve();
}).then(() => {
  console.log(3);
});
process.nextTick(() => {
  console.log(2);
});
console.log(1);
```

これの実行結果は以下のようになります。

```tsx
1;
2;
3;
```

何度行っても同じ結果になります。
今回解説する DataLoader は process.nextTick で登録した関数の中で Promise の resolve を呼び出しているため、この順番が担保されるのはとても重要となります。

### 登場人物 ② Promise

Javascript 関係の言語を触っていると目にしない日はないくらいよく使う Promise ですが、今回も大切になってきます。
ただ、今回把握しておいて欲しいことが一点あります。
それは以下の点です。
**resolve,reject 関数は Promise の外で呼び出しても処理が完了する**
具体例を示します。
まずはよく見る resolve 関数の呼び出しです。

```tsx
const promise = new Promise((resolve) => {
  resolve("Promiseの中で実行");
});
promise.then((res) => console.log(res));
//出力値
//Promiseの中で実行
```

Promise の中で resolve を呼び出し、そこに値をセットしています。
これによって、promise が成功したときはセットした値を受け取ることができます。
次に以下のサンプルコードをみてください。

```tsx
const a = [];
const promise = new Promise((resolve) => {
  a.push({ resolve });
});
a[0].resolve("Promiseの外で実行");
promise.then((res) => console.log(res));
//出力値
//Promiseの外で実行
```

Promise の中で resolve 関数は呼び出さず、変数 a 配列に追加してます。
そして配列から resolve 関数を呼び出しています。
これでも resolve には値が渡るので、成功時の処理で値を受け取ることができています。
Promise は他にも機能を持っていますが、今回はこの resolve 関数を実行する場所は Promise の外でもよいことは認識しておいてください。
以上必要な事前準備が完了したので、早速 load メソッドを追っていきます。

## load メソッドは何をしているか

では load メソッドの流れを見ていきます。
なお、本題に入る前に以下の点は前提としているので、ご注意ください。
①options はすべて未指定
②Node.js 環境下で実行(=process.nextTick が使用できる)
③ キャッシュ周りは今回無視する
④ エラーハンドリングについては扱わない
でははじめます。

### load メソッドの全体像

まずは load メソッドを有している DataLoader クラスの[コンストラクタ](https://github.com/graphql/dataloader/blob/main/src/index.js#L47)を見てみましょう。

```tsx
class DataLoader<K, V, C = K> {
  constructor(batchLoadFn: BatchLoadFn<K, V>, options?: Options<K, V, C>) {
    if (typeof batchLoadFn !== "function") {
      throw new TypeError(
        "DataLoader must be constructed with a function which accepts " +
          `Array<key> and returns Promise<Array<value>>, but got: ${batchLoadFn}.`
      );
    }
    this._batchLoadFn = batchLoadFn;
    this._batchScheduleFn = getValidBatchScheduleFn(options);
    //...略
  }
  //...略
}
```

引数で設定した関数を\_batchLoadFn プロパティに格納しています。
また、\_batchScheduleFn プロパティには getValidBatchScheduleFn 関数の戻り値を格納しています。
では、getValidBatchScheduleFn 関数の中身を見ていきます。

```tsx
function getValidBatchScheduleFn(
  options: ?Options<any, any, any>,
): (() => void) => void {
  const batchScheduleFn = options && options.batchScheduleFn;
  if (batchScheduleFn === undefined) {
    return enqueuePostPromiseJob;
  }
  if (typeof batchScheduleFn !== 'function') {
    throw new TypeError(
      `batchScheduleFn must be a function: ${(batchScheduleFn: any)}`,
    );
  }
  return batchScheduleFn;
}
```

今回は options の指定はない前提なので、実質処理は以下の通りです。

```tsx
function getValidBatchScheduleFn(
  options: ?Options<any, any, any>,
): (() => void) => void {
    return enqueuePostPromiseJob;
}
```

変数 enqueuePostPromiseJob を返す関数になりました。
変数 enqueuePostPromiseJob は以下のコードです。

```tsx
const enqueuePostPromiseJob =
  typeof process === "object" && typeof process.nextTick === "function"
    ? function (fn) {
        if (!resolvedPromise) {
          resolvedPromise = Promise.resolve();
        }
        resolvedPromise.then(() => {
          process.nextTick(fn);
        });
      }
    : typeof setImmediate === "function"
    ? function (fn) {
        setImmediate(fn);
      }
    : function (fn) {
        setTimeout(fn);
      };
```

Node.js で動かす想定なので、この変数は今回以下の形とみなせます。

```tsx
const enqueuePostPromiseJob = function (fn) {
  if (!resolvedPromise) {
    resolvedPromise = Promise.resolve();
  }
  resolvedPromise.then(() => {
    process.nextTick(fn);
  });
};
```

引数を process.nextTick に設定していますね。
なお、ここで Promise.resolve を設定していますが、実はこれの必要性はよくわかっていません。
なくても動くと思うのですが、なぜここにわざわざ Promise を設定し、実行を後回しにしているのかは調べてもよくわかりませんでした。
なので、知っている方がいれば是非とも教えてください。
話をもとに戻します。
これまでのことから\_batchScheduleFn プロパティは以下のように見なせそうです。

```tsx
this._batchScheduleFn ≒ (fn) => process.nextTick(fn)
```

load メソッドで実行で使うプロパティの準備はできたので、[load メソッド](https://github.com/graphql/dataloader/blob/main/src/index.js#L74C1-L112C4)
の中身を見ていきます。

```tsx
  load(key: K): Promise<V> {
    if (key === null || key === undefined) {
      throw new TypeError(
        'The loader.load() function must be called with a value, ' +
          `but got: ${String(key)}.`,
      );
    }
    const batch = getCurrentBatch(this);
    const cacheMap = this._cacheMap;
    const cacheKey = this._cacheKeyFn(key);
    // If caching and there is a cache-hit, return cached Promise.
    if (cacheMap) {
      const cachedPromise = cacheMap.get(cacheKey);
      if (cachedPromise) {
        const cacheHits = batch.cacheHits || (batch.cacheHits = []);
        return new Promise(resolve => {
          cacheHits.push(() => {
            resolve(cachedPromise);
          });
        });
      }
    }
    // Otherwise, produce a new Promise for this key, and enqueue it to be
    // dispatched along with the current batch.
    batch.keys.push(key);
    const promise = new Promise((resolve, reject) => {
      batch.callbacks.push({ resolve, reject });
    });
    // If caching, cache this promise.
    if (cacheMap) {
      cacheMap.set(cacheKey, promise);
    }
    return promise;
  }
```

キャッシュは今回話題にしませんし、エラーハンドリングを無視すると以下のようになります。

```tsx
load(key: K): Promise<V> {
    const batch = getCurrentBatch(this)
    batch.keys.push(key);
    const promise = new Promise((resolve, reject) => {
      batch.callbacks.push({ resolve, reject });
    });
    return promise;
  }
```

大分短くなりましたね。
次に、getCurrentBatch 関数の中身を見ていきます。

```tsx
function getCurrentBatch<K, V>(loader: DataLoader<K, V, any>): Batch<K, V> {
  // If there is an existing batch which has not yet dispatched and is within
  // the limit of the batch size, then return it.
  const existingBatch = loader._batch;
  if (
    existingBatch !== null &&
    !existingBatch.hasDispatched &&
    existingBatch.keys.length < loader._maxBatchSize
  ) {
    return existingBatch;
  }
  // Otherwise, create a new batch for this loader.
  const newBatch = { hasDispatched: false, keys: [], callbacks: [] };
  // Store it on the loader so it may be reused.
  loader._batch = newBatch;
  // Then schedule a task to dispatch this batch of requests.
  loader._batchScheduleFn(() => {
    dispatchBatch(loader, newBatch);
  });
  return newBatch;
}
```

ちょっと分かりにくいかもしれませんが、大きく分けて以下の二つのことをやっています。
①`{ hasDispatched: boolean, keys: string[], callbacks: string[] }`のオブジェクトを Dataloader の\_batch プロパティに格納し、その値を返す。
②process.nextTick に dispatchBatch 関数を登録している
本丸の dispatchBatch 関数の処理に行く前に load メソッドの中身をこれまでのことを反映させたものに置き換えます。

```tsx
class DataLoader {
  _batch = { hasDispatched: false, keys: [], callbacks: [] };
  load(key: K): Promise<V> {
    if (!this._batch.hasDispatched) {
      this._batch.hasDispatched = true;
      process.nextTick(dispatchBatch(this, this._batch));
    }

    this._batch.keys.push(key);
    const promise = new Promise((resolve, reject) => {
      this._batch.callbacks.push({ resolve, reject });
    });

    return promise;
  }
}
```

load メソッドが呼ばれた最初だけ process.nextTick に dispatchBatch 関数を登録し、それ以降は\_batch プロパティにキーと Promise の resolve,reject 関数を渡しています。
最後に Promise を返しています。
ここまでの流れと同期・非同期処理の実行順番を考えると少しづつ、load メソッドを呼ばれ続ける間は\_batch プロパティに値を入れ続けているだけということが見えてきます。
では、\_batch プロパティに値をつめ終えた後の処理、すなわち dispatchBatch 関数の中身を見ていきます。
なお、今回は最初から解説に関わる部分しか提示しませんので、全てのコードが見たい場合は[ソースコード](https://github.com/graphql/dataloader/blob/main/src/index.js#L299C1-L380C2)を確認してください。
それでは dispatchBatch 関数の処理の一部を展開します。

```tsx
function dispatchBatch<K, V>(
  loader: DataLoader<K, V, any>,
  batch: Batch<K, V>
) {
  let batchPromise;
  try {
    batchPromise = loader._batchLoadFn(batch.keys);
  } catch (e) {
    //...略
  }
  batchPromise
    .then((values) => {
      // Step through values, resolving or rejecting each Promise in the batch.
      for (let i = 0; i < batch.callbacks.length; i++) {
        const value = values[i];
        if (value instanceof Error) {
          batch.callbacks[i].reject(value);
        } else {
          batch.callbacks[i].resolve(value);
        }
      }
    })
    .catch((error) => {
      //...略
    });
}
```

\_batchLoadFn プロパティはコンストラクタで定義した関数で、その関数の引数に keys プロパティの配列を渡していますね。
後はコンストラクタの関数から受け取った値を、callbacks プロパティのインデックス番号と一致するものを取り出し、値があれば resolve 関数に渡しています。
これによって、先程 load メソッドで定義した Promise がペンディング状態から完了に代わり、値を受け取ります。
resolve 関数に値を渡すのは同期的に行われるので、load メソッドは一つづつ値を返します。
以上が load メソッドの流れです。
では最後に load メソッドを改変して、全ての処理を load メソッドの中に閉じ込めてみましょう。

```tsx
class DataLoader {
  _batch = { hasDispatched: false, keys: [], callbacks: [] };
  constructor(batchLoadFn) {
    this._batchLoadFn = batchLoadFn;
  }
  load(key: K): Promise<V> {
    if (!this._batch.hasDispatched) {
      this._batch.hasDispatched = true;
      process.nextTick(() => {
        const batchPromise = loader._batchLoadFn(batch.keys);
        batchPromise.then((values) => {
          // Step through values, resolving or rejecting each Promise in the batch.
          for (let i = 0; i < batch.callbacks.length; i++) {
            const value = values[i];
            if (value instanceof Error) {
              batch.callbacks[i].reject(value);
            } else {
              batch.callbacks[i].resolve(value);
            }
          }
        });
      });
    }
    this._batch.keys.push(key);
    const promise = new Promise((resolve, reject) => {
      this._batch.callbacks.push({ resolve, reject });
    });
    return promise;
  }
}
```

大分スッキリ？しましたね。
中々腑に落ちにくいとは思いますが、process.nextTick は同期処理が完了した後に実行されるのと、Promise の resolve,reject 関数はどこで呼び出したとしても Promise に値がセットされるという二点を把握してください。
それを頭に入れて、最初は同期処理でキーと関数の登録を行っている、登録が終わったら process.nextTick が実行され、その中で関数が起動する理解してください。
何となくにはなりますが、以前よりは腑に落ちるかと思います。

## おわりに

今回は DataLoader クラスの load メソッドがどのような動きをするのかを見ていきました。
この記事を書いて思ったのが、この処理賢いなということです。
今の自分はこんな実装方法全く思い浮かびません。
たった一ファイルを理解するのにかなり時間を要してしまいましたが、Javascript の奥深い世界を少しだけ除けたので満足です。
ここまで読んでいただきありがとうございました。
