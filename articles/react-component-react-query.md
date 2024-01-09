---
title: "組み込みコンポーネントとReact Query"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React", "ReactQuery"]
published: true
---

## はじめに

React には様々な機能があります。
フックやコンポーネント、イベント設定などなど…。
あげれば切りがないですが、今回は Suspense と Error Boundary のコンポーネントを中心に見ていきます。
そして、これら組み込みコンポーネントと React Query を使用して連携する方法についても見ていきます。
軽くしか触れていませんが、それでも中々便利だと思ったので読んでいただければ幸いです。

## Error Boundary コンポーネント

### 概要

React は特に何もしない場合、エラーが発生すると画面に要素が何も表示されなくなる可能性があります。
例えば以下のようなコードを作成します。

```tsx
export default function ErrorBoundaryBasic() {
  throw Error("エラーです");
  return <p>エラーテスト</p>;
}
```

これを別のファイルで呼び出します。

```tsx
<>
  <ErrorBoundaryBasic></ErrorBoundaryBasic>
  <p>テストです</p>
  <p>表示されるかテストです</p>
</>
```

この状態で画面を表示させると、以下のようにエラーメッセージが表示され、画面は何も描画していません。
![2024-01-06_10h30_55.png](/images/react-component-react-query/2024-01-06_10h30_55.png)
このように、断片的な問題であったとしてもアプリ全体が停止してしまいます。
その機能がないとアプリケーションの根幹に関わるものなら、上記状態になってしまっても問題ありません。
要素だけ表示させたとしても、アプリの目的を達成できないからです。
しかし、アプリ全体の挙動に関わるものではないなら、エラーが発生してもアプリ全体まで止まってほしくはありません。
該当領域を無効化などするだけに留めたいです。
そこで使うのが Error Boundary です。
Error Boundary は配下のコンポーネントで発生したエラーを補足&記録して、代わりの要素を表示させることができます。
これによって、アプリ全体の動きを止めずエラー周りの管理を行うことができます。
Error Boundary コンポーネントの利点が分かったので、早速使ってみます。

### 使ってみる

[ドキュメント](https://ja.react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)を見ると React で Error Boundary コンポーネントを実装できそうですが、ドキュメントに記載されている方法はクラスコンポーネントが前提となっています。
現在 React は関数コンポーネントで書くのが主流なので、ドキュメントの方法をそのまま使用することはできません。
しかし、[ドキュメント](https://ja.react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary:~:text=%E3%81%8C%E3%81%82%E3%82%8A%E3%81%BE%E3%81%9B%E3%82%93%E3%80%82-,%E8%A3%9C%E8%B6%B3,%E3%81%AB%20react%2Derror%2Dboundary%20%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%99%E3%82%8B%E3%81%93%E3%81%A8%E3%81%8C%E3%81%A7%E3%81%8D%E3%81%BE%E3%81%99%E3%80%82,-%E4%BB%A3%E6%9B%BF%E6%A1%88)にもあるように[react-error-boundary](https://github.com/bvaughn/react-error-boundary)をインストールすれば関数コンポーネントでも、Error Boundary コンポーネントを使うことができます。
なので、まずは`npm install react-error-boundary` でモジュールをインストールします。
次に、概要で示した ErrorBoundaryBasic コンポーネントを呼び出しいる部分を以下のように変更します。

```tsx
<>
  <ErrorBoundary fallback={<p>エラーが起きています</p>}>
    <ErrorBoundaryBasic></ErrorBoundaryBasic>
  </ErrorBoundary>
  <p>テストです</p>
  <p>表示されるかテストです</p>
</>
```

このようにして、画面を再度表示させると相変わらずエラーは発生していますが、ErrorBoundaryBasic コンポーネント部分は fallback で設定した要素が表示されています。
さらに、その他の要素はそのまま表示されています。
![2024-01-06_10h42_55.png](/images/react-component-react-query/2024-01-06_10h42_55.png)
よって、Error Boundary コンポーネント内で発生したエラーは Error Boundary コンポーネント内で完結していることが分かります。
Error Boundary コンポーネントの基礎的な使い方はわかりましたが、エラー発生時に要素の表示だけでなく、何かしらの処理を搭載したい場合は往々にしてあります。
なので、fallbackRender 属性を使用して上記要望を達成する例を見ていきます。
Error Boundary コンポーネントを呼び出している部分を以下のように変更します。

```tsx
import ErrorBoundaryBasic from "./ErrorBoundaryBasic";
import { ErrorBoundary, FallbackProps } from "react-error-boundary";
function App() {
  const handleFallback = ({ error, resetErrorBoundary }: FallbackProps) => {
    const handleClick = () => resetErrorBoundary();
    return (
      <div>
        <h3>以下のエラーが発生しました</h3>
        <p>{error.message}</p>
        <button type="button" onClick={handleClick}>
          リトライ
        </button>
      </div>
    );
  };
  return (
    <>
      <ErrorBoundary
        fallbackRender={handleFallback}
        onReset={() => alert("リセットです")}
      >
        <ErrorBoundaryBasic></ErrorBoundaryBasic>
      </ErrorBoundary>
      <p>テストです</p>
      <p>表示されるかテストです</p>
    </>
  );
}
export default App;
```

注目するのは fallbackRender 属性です。
fallbackRender 属性は型定義ファイルを辿ると、以下の関数の typeof となっています。

```tsx
declare function FallbackRender(props: FallbackProps): ReactNode;
```

そして、FallbackProps 型は以下通りです。

```tsx
export type FallbackProps = {
  error: any;
  resetErrorBoundary: (...args: any[]) => void;
};
```

error は受け取ったエラーであり、resetErrorBoundary は Error Boundary 内の内容を再描画する関数です。
以上のことから、fallbackRender 属性には画面要素を返す関数を渡せば良いので、関数内で実行する処理は任意の機能を実装できます。
なお、resetErrorBoundary 関数を実行したときは、Error Boundary コンポーネントの onReset 属性で補足できるので、再描画させるときに何かしらの関数を実行したい場合は、onReset 属性に設定してください。
fallbackRender 属性を見ていると、コンポーネントを突っ込んでいるみたいだと感じます。
なら、fallbackRender 属性にコンポーネントを設定すれば同様に表示できそうです。
しかし、Error Boundary コンポーネントはコンポーネントを設定するときは、別途 FallbackComponent 属性があるのでそちらを使用します。
書き方は fallbackRender 属性の時に示した例とほぼ一緒です。
handleFallback 関数の処理をコンポーネントとして切り出して、そのコンポーネントを FallbackComponent 属性に設定するだけで良いです。
ここまでで、Error Boundary コンポーネントの使い方を見てきましたが、実はどれもレンダリングの際のエラーについてでした。
これは Error Boundary コンポーネントが、あくまでレンダリング過程でのエラーを補足するもので、イベントハンドラーからの例外は補足の対象外となっているためです。
なので、イベントハンドラーのエラーも補足するには追加で機能が必要です。
最後にその機能について見ていきます。
ただ、イベントハンドラーを補足するための実装は Error Boundary コンポーネント側で行いません。
代わりに、Error Boundary コンポーネント内の内部で補足できるような処理を搭載します。
具体的には以下のように useErrorBoundary 関数を使用します。

```tsx
import { useErrorBoundary } from "react-error-boundary";
export default function ErrorBoundaryBasic() {
  const { showBoundary } = useErrorBoundary();
  const handleClick = () => {
    try {
      throw Error("エラーです");
    } catch (e) {
      showBoundary(e);
    }
  };
  return <button onClick={handleClick}>ボタン</button>;
}
```

useErrorBoundary 関数から取得できる showBoundary 関数をエラー発生ときに実行すれば、Error Boundary コンポーネントはエラーを補足でき、代替要素を表示させることが可能となります。
以上が Error Boundary コンポーネント周りの解説です。
次に、Suspense コンポーネントについてみていきます。

## Suspense コンポーネント

### 概要

Suspense コンポーネントは[ドキュメント](https://react.dev/reference/react/Suspense)を確認すると以下のように記載されています。

> `<Suspense>` lets you display a fallback until its children have finished loading.

子要素のローディングが終わるまで、fallback で設定したものを表示させると書かれています。
なんとなく、描画を待ってくれるものだと分かります。
コンポーネントによっては、描画する際に外部から情報を取得してから行いたいものがあります。
また、容量の大きいなど何かしらの事情でコンポーネント自体を読み込むタイミングを遅延させたい場合もあります。
上記のような状況で、コンポーネントを読み込んでいるときに何も表示させないと、ユーザーとしては何が行われているか分からなくなってしまいます。
これでは、アプリケーションの使い勝手がよくありません。
そこで、Suspense を使います。
Suspense を使うことで、ローディング中に何かしらの要素を表示させることができます。
これによって、ユーザーは画面の読み込み中だとわかり、安心して読み込みを待つことができます。
サラッと Suspense の概要と利点を説明したので、次は実際に使用してみます。

### Suspense を使ってみる

ここでは実際に Suspense コンポーネントを使用してみます。
まずはコンポーネントの遅延ロードの場合です。
遅延ロードさせる任意のコンポーネントを作成してください。
今回は以下のような LazyButton コンポーネントを作成しました。

```tsx
// LazyButton.tsx
export default function LazyButton() {
  return (
    <>
      <button>ボタン</button>
    </>
  );
}
```

次に上記コンポーネントを遅延ロードさせる処理を記載したコンポーネントを作成します。

```tsx
import { Suspense, lazy } from "react";
const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));
const LazyButton = lazy(() => sleep(2000).then(() => import("./LazyButton")));
const LazyButton2 = lazy(() => sleep(1000).then(() => import("./LazyButton")));
export default function LazyBasic() {
  return (
    <Suspense fallback={<p>Now Loading</p>}>
      <LazyButton></LazyButton>
      <LazyButton2></LazyButton2>
    </Suspense>
  );
}
```

こうすれば、最初画面上では「Now Loading」の文言が表示され、2 秒後にボタンが二つ表示されます。
コードについても見ていきましょう。
sleep 関数については、意図的に待たせるための関数なので実際の開発で使用することはそこまでないと思います。
注目して欲しいのは lazy 関数と Suspense コンポーネントです。
まず[lazy 関数](https://react.dev/reference/react/lazy)についてみていきます。
lazy 関数は以下のように記載します。

```tsx
lazy(() => 戻り値がReactコンポーネントであるPromise);
```

具体的には先程書いたように、`lazy(()⇒import(’コンポーネントパス’))`といったようなコンポーネントを読み込む処理を戻り値にした関数をパラメータとして渡します。
このようにすれば、パラメータの関数の処理が適切に完了した際コンポーネントを取得することができます。
後は lazy 関数の戻り値を変数として受け取り、コンポーネントの戻り値に記載することで遅延ロードを行うコンポーネントが設定できます。
遅延ロードしたいコンポーネントは lazy 関数で読み込むだけで良いのですが、使用する際は二つの注意点があります。
まずは lazy 関数を記載する場所で、[コンポーネントを定義している関数の外に記載します](https://react.dev/reference/react/lazy#troubleshooting)。
lazy 関数は[ドキュメント](https://react.dev/reference/react/lazy#parameters)にも記載があるように、コンポーネントを取得した場合その値をキャッシュします。
それによって、lazy 関数は一度実行すれば良くなります。
しかし、コンポーネントを定義する関数の中に記載してしまうと、コンポーネントが再描画されるタイミングで常に実行されてしまいます。
一応動くは動くのですが、lazy 関数の特性を打ち消してしまっているのでコンポーネントの外で lazy 関数は呼び出すようにします。
二つ目の注意点は lazy 関数というより Suspense の話ですが、lazy 関数で取得したコンポーネントの描画タイミングについてです。
先程提示したサンプルコードでは、二つのボタンコンポーネントを Suspense 内に記載していました。
片方は 1 秒後にコンポーネントを取得しており、もう片方は 2 秒後にコンポーネントを取得するようにしていました。
なので、直感的には 1 秒後にボタンが一つ表示され、さらに 1 秒後にもう一つのボタンが表示されると思います。
しかし、実際は 2 秒後にボタンが二つ表示されています。
これは Suspense の特性で、Suspense コンポーネントは囲まれた要素が全てロードされたタイミングで描画するようにします。
そのため、仮に lazy 関数でそれぞれのロード時間をずらしても順々に表示されません。
この特性を把握しておかないと、lazy 関数で明示的にずらしてもいるのになぜか同時に表示されていると混乱しそうだったので、記載しました。
なお、[ドキュメント](https://react.dev/reference/react/Suspense#revealing-nested-content-as-it-loads)には Suspense を階層で記載すれば、順々でローディングする例も示されています。
今回のコードで言うと以下のような書き方をすれば、Now Loading→ ボタン一つ+「Now Loading2」→ ボタン二つという順番で表示されます。

```tsx
import { Suspense, lazy } from "react";
const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));
const LazyButton = lazy(() => sleep(2000).then(() => import("./LazyButton")));
const LazyButton2 = lazy(() => sleep(1000).then(() => import("./LazyButton")));
export default function LazyBasic() {
  return (
    <Suspense fallback={<p>Now Loading</p>}>
      <Suspense fallback={<p>Now Loading2</p>}>
        <LazyButton></LazyButton>
      </Suspense>
      <LazyButton2></LazyButton2>
    </Suspense>
  );
}
```

ここまで、lazy 関数中心で見てきたので、次は Suspense コンポーネントについてです。
Suspense コンポーネントは以下のように記載します。

```tsx
<Suspense fallback={ローディング中に表示させる要素}>
  <SomeComponent />
</Suspense>
```

今回の例では fallback に HTML 要素を直接記載していますが、もちろん React コンポーネントを記載しても問題ありません。
Suspense コンポーネントの中には、実際に描画させたいコンポーネントを記載します。
正直遅延ロードについては、Suspense コンポーネント周りの説明は以上となります。
ただ、非同期処理での Suspense コンポーネントは少し特殊な部分があるので、別途見ていきます。
非同期処理(Promise)を受け取って処理が完了するまで Suspense の fallback を表示させるには、非同期処理を行っている側で Promise を throw するようにします。

```tsx
export default function ThrowPromise() {
  throw new Promise((resolve, reject) => {});
}
```

throw するのは大体例外処理の時なので、Promise を throw するのは違和感があると思います。
ただ、Suspense は子要素から投げられた Promise を補足した時のみ、fallback の要素を表示するのでこのことは覚えておく必要があります。
Suspense が Promise を補足する方法については分かったので、Promise 処理が終わった時に要素を表示させる場合についても見ていきます。
Promise は resoleve もしくは、reject が実行されることで処理が完了します。
なので、要素を表示させる場合も同様にすれば良いです。
例として、先程作成した ThrowPromise 関数を少し変更します。

```tsx
let flag = false;
export default function ThrowPromise() {
  if (flag) {
    return <p>完了</p>;
  }
  throw new Promise((resolve) => {
    setTimeout(() => {
      flag = true;
      resolve("Success");
    }, 2000);
  });
}
```

2 秒後に要素を表示させる flag を true にして、resolve を実行しています。
後はこのコンポーネントを以下のように Suspense で囲みます。

```tsx
<Suspense fallback={<p>Now Loading</p>}>
  <ThrowPromise />
</Suspense>
```

Suspense は Promise の処理が完了したタイミングで、改めてコンポーネントを描画します。
そのため、Promise を throw された時は「Now Loading」が表示され、2 秒後に resolve が実行されるのでコンポーネントを描画しようとします。
その時は flag は true になっているので、「完了」という文言が画面に表示されています。
このように、非同期処理で Suspense を使う場合、Promise を投げることで Promise 内の処理が完了するまで、fallback を表示させるという関係性がわかりました。
しかし、処理が完了という部分で注意が必要です。
Suspense は Promise の処理が完了さえすれば役目を終えます。
そのため、先程示した ThorwPromise 関数内の resolve 部分を`reject('Error')` と変更したとしても、Suspense の処理自体は完了します。
よって、reject の場合はエラーを発生させるようにしたい場合は以下のようなラッパーを作成する必要があります。

```tsx
export default function wrapPromise<T>(promise: Promise<T>) {
  var status: "pending" | "reject" | "fulffilled" = "pending";
  let data: T;
  const wrapper = promise
    .then((res) => {
      status = "fulffilled";
      data = res;
    })
    .catch((e) => {
      status = "reject";
      data = e;
    });
  return {
    get() {
      if (status === "fulffilled") {
        return data;
      }
      if (status === "reject") {
        throw data;
      }
      throw wrapper;
    },
  };
}
```

後はこのラッパーを使ったコンポーネントを作成して、それを Suspense で囲めば、エラーの際例外を投げ、成功した時のみ要素が表示されるようになります。

```tsx
import wrapPromise from "./WrapPromise";
const info = getInfo();
export default function Result() {
  const result = info.get();
  return <p>{result}</p>;
}
function getInfo() {
  return wrapPromise<string>(
    new Promise((resolve, reject) => {
      setTimeout(() => {
        if (Math.random() > 0.5) {
          resolve("aaaaaa");
        } else {
          reject(Error("エラーです"));
        }
      }, 2000);
    })
  );
}
//上記コンポーネントを以下のようにSuspenseで囲む
<Suspense fallback={<p>Now Loading...</p>}>
  <Result />
</Suspense>;
```

以上が Suspense の説明となります。
遅延ロード周りについては、それなりにこのまま使用するかもしれませんが、非同期処理周りについては正直このまま使用することはないと思います。
というのも、React には React Query や Next.js といったライブラリ/フレームワークでデータを取得することが多いため、自分で非同期処理をガリガリ書くことはあまりないです。
なので、ここからは Reaqt Query の概要を説明した後、React Query と Suspense、加えて Error Boundary の連携を行っていきます。

## React Query

### 概要

ここでは React Query の概要を説明します。
と言いたいところですが、正直以下の良記事達で事足りるので概要については、それら記事にぶん投げます。
[React Query はデータフェッチライブラリではない。非同期の状態管理ライブラリだ。](https://qiita.com/taisei-13046/items/05cac3a2b4daeced64aa)
[React Query-基礎知識編](https://zenn.dev/akineko/articles/8c047c3473aed0)

### 使ってみる

まずは`npm install react-query`でモジュールをインストールします。
その後、なるべく上位コンポーネント(今回は App.tsx)で以下のように、Provider を設定します。

```tsx
import "./App.css";
import { QueryClient, QueryClientProvider } from "react-query";
import ReactQueryBasic from "./ReactQueryBasic";
const cli = new QueryClient();
function App() {
  return (
    <QueryClientProvider client={cli}>
      <ReactQueryBasic></ReactQueryBasic>
    </QueryClientProvider>
  );
}
export default App;
```

クライアントとして渡すために QueryClien をインスタンス化した後、QueryClientProvider コンポーネントに渡します。
これによって、各コンポーネントで React Query を使用する準備ができました。
それでは、ReactQueryBasic コンポーネントを以下のように作成します。

```tsx
import { useQuery } from "react-query";
const sleep = (delay: number) =>
  new Promise((resolve) => setTimeout(resolve, delay));
const fetchJson = async () => {
  await sleep(2000);
  const id = Math.floor(Math.random() * 301);
  try {
    const res = await fetch(`https://jsonplaceholder.typicode.com/todos/${id}`);
    if (res.status === 404) {
      throw new Error("TODOが見つかりませんでした");
    }
    if (res.ok) {
      return res.json();
    }
  } catch (e) {
    if (e instanceof Error) {
      throw e;
    }
    throw new Error("予期せぬエラーです");
  }
};
export default function ReactQueryBasic() {
  const { data, isLoading, isError, error } = useQuery("json", fetchJson);
  if (isLoading) {
    return <p>Loading...</p>;
  }
  if (isError) {
    return (
      <p>
        Error:{" "}
        {error instanceof Error
          ? error.message
          : "予期せぬエラーが発生しました"}
      </p>
    );
  }
  return (
    <div>
      <p>TODO名:{data.title}</p>
      <p>完了有無:{data.completed ? "完了" : "未完了"}</p>
    </div>
  );
}
```

今回は外部データを[JSONPlacefolder](https://jsonplaceholder.typicode.com)から fetch で取得するようにしています。
そして、0 から 300 までの id をランダムで設定し、その ID と一致する JSON データを取得するようにしています。
データが存在すればその内容を表示させ、無ければエラーメッセージを表示させます。
JSONPlaceholder と通信中であれば、ローディングメッセージが表示されます。
useEffect とか使用せずにできるので、中々便利ですね。
なお、上記サンプルについては二点注意点があります。
一つ目は fetch による通信です。
今回は fetch で外部と通信していますが、場合によっては axios などで通信することは多々あります。
axios の場合は以下のコードを書くだけで、例外発生時 useQuery の isError などのエラー周りは検知してくれます。

```tsx
const res = await axios.get(
  `https://jsonplaceholder.typicode.com/todos/存在しないID`
);
return res.data;
```

一方で、fetch 関数はエラー自体は返ってくるのですが、それを useQuery が補足できません。
そのため、fetch を使って通信をする場合、明示的にエラー処理を書く必要があります。
二つ目の注意点は、エラー時の useQuey の再実行についてです。
実は今回示したサンプルでは中々エラーメッセージが画面に表示されることはありません。
これは[ドキュメント](https://tanstack.com/query/latest/docs/react/guides/query-retries)にもあるように、React Query は通信が失敗したとき自動で通信を再実行します。
デフォルトの値が 3 となっているので、3 回存在しない ID で通信しない限りエラーメッセージは表示されません。
私はこの特性を知らず、1 時間ぐらい何で上手くエラーメッセージが表示されないのか悩んだので共有しました。
以上が React Query の簡単な使い方の解説です。
ようやく下準備が完了したので、本題の組み込みコンポーネントと React Query を合わせて使ってみます。

## React Query と組み込みコンポーネントを連携する

最後にこれまで見てきた React Query と Suspense や Error Boundary を組み合わせてみます。
といっても、Promise を投げるのは React Query が良しなにやってくれますので、あまり作業としてはありません。
早速やってみましょう。
useQuery を使用しているコンポーネントは以下のように変更します。

```tsx
import { useQuery } from "react-query";
const sleep = (delay: number) =>
  new Promise((resolve) => setTimeout(resolve, delay));
const fetchJson = async () => {
  await sleep(2000);
  const id = Math.floor(Math.random() * 301);
  try {
    const res = await fetch(`https://jsonplaceholder.typicode.com/todos/${id}`);
    if (res.ok) {
      return res.json();
    }
  } catch (e) {
    if (e instanceof Error) {
      throw e;
    }
    throw new Error("予期せぬエラーです");
  }
};
export default function ReactQueryBasic() {
  const { data } = useQuery("json", fetchJson);
  return (
    <div>
      <p>TODO名:{data.title}</p>
      <p>完了有無:{data.completed ? "完了" : "未完了"}</p>
    </div>
  );
}
```

先程記載していたローディング中とエラー時の分岐を削除しています。
次に、削除したローディングとエラーを補足できるように、上記コンポーネントの呼び出し元を以下のようにします。

```tsx
<Suspense fallback={<p>Loading</p>}>
  <ErrorBoundary fallback={<p>エラー発生</p>}>
    <QueryClientProvider client={cli}>
      <ReactQueryBasic></ReactQueryBasic>
    </QueryClientProvider>
  </ErrorBoundary>
</Suspense>
```

ただし、QueryClient のオプションは以下のように Suspense を有効にする必要があります。

```tsx
const cli = new QueryClient({
  defaultOptions: {
    queries: {
      suspense: true,
    },
  },
});
```

以上で、useQuery を使用しているコンポーネントにローディングや、エラー時の分岐を記載せずに対応できるようになりました。
もちろんこれが最適というわけではないですが、コンポーネントごとにいちいちエラー時やローディングの設定をしなくてもよくなるのは便利ですね。
外部通信が多い要素などを Suspense で囲むとかは結構使い道ありそうです。
動作確認を行うと、最初は「Loading」の文字が表示され、ID と一致する TODO が見つからない時は「エラー発生」が表示されるのを確認できると思います。

## おわりに

今回は Suspense、Error Boundary といった組み込みコンポーネントと外部通信モジュールである React Query について見ていきました。
なんとなく感覚で使っていたがゆえに、いまいち活用どころがわからないとおもっていたのが、記事にすることでちゃんと使えば便利だと感じました。
今後 React を触れる機会は増えそうなので、こういった便利グッズは有効活用したいです。
また、React Query はまだまだ奥が深そうなので、別途記事として取り上げることができたらと思います。
ここまで読んでいただきありがとうございました。
