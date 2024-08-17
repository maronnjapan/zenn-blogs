---
title: "Reactのchildrenが想定と異なる挙動をするなと思ったら、Reactがどうコンパイルされるかがよくわかっていなかった話"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React", "Babel", "JSX"]
published: true
---

## はじめに

まず以下のコードをご覧ください。

```tsx
function Test({ children }: { children: ReactNode }) {
  return <>{typeof children === "string" ? <p>{children}</p> : null}</>;
}
function Test2() {
  const text = "test";
  return <Test>value:{text}</Test>;
}
```

このコード内にある Test2 コンポーネントを呼び出した時、画面はどのようになるでしょうか？
正解は何も表示されません。
今回はそうなる挙動についてみていきます。
注意点として、この記事はあくまで出力される結果ベースでの話です。
React がなぜその挙動をするかまではカバーできていませんので、ご了承ください。
では始めます。

## JSX とは

[React のドキュメント](https://ja.react.dev/learn/writing-markup-with-jsx)を見てみると以下の記載があります。

> _JSX_ とは JavaScript の拡張であり、JavaScript ファイル内に HTML のようなマークアップを書けるようにするものです。

React で、html での要素を当たり前に書いていますが、よくよく考えたら不思議ですね。
その不思議を吸収してくれるのが JSX になります。
この JSX のおかげで、Javascript のコードの中で HTML ライクの書き方で画面を定義できます。
なので、React 公式も基本的に JSX でのコンポーネントを記載することを推奨しています。
ちなみにですが、JSX 自体は[React のドキュメント](https://ja.react.dev/learn/writing-markup-with-jsx#:~:text=JSX%20%E3%81%A8%20React%20%E3%81%AF%E5%88%A5%E3%81%AE%E7%89%A9%E3%81%A7%E3%81%99%E3%80%82%E4%B8%80%E7%B7%92%E3%81%AB%E4%BD%BF%E3%82%8F%E3%82%8C%E3%82%8B%E3%81%93%E3%81%A8%E3%81%8C%E5%A4%9A%E3%81%84%E3%81%A7%E3%81%99%E3%81%8C%E3%80%81%E7%89%87%E6%96%B9%E3%81%A0%E3%81%91%E3%82%92%E7%8B%AC%E7%AB%8B%E3%81%97%E3%81%A6%E4%BD%BF%E3%81%86%E3%81%93%E3%81%A8%E3%81%AF%E5%8F%AF%E8%83%BD%E3%81%A7%E3%81%99%E3%80%82JSX%20%E3%81%A8%E3%81%AF%E8%A8%80%E8%AA%9E%E3%81%AE%E6%8B%A1%E5%BC%B5%E3%81%A7%E3%81%82%E3%82%8A%E3%80%81React%20%E3%81%AF%20JavaScript%20%E3%83%A9%E3%82%A4%E3%83%96%E3%83%A9%E3%83%AA%E3%81%A7%E3%81%99%E3%80%82)に書いてある通り、React の一機能ではありません。
あくまで、React は JSX を使用して要素を記述できますが、JSX は React に限ったものではありません。

## コンパイル結果を見る

JSX について本当に軽く見ていきました。
JSX は HTML を書く時とかなり似ているので、コンポーネント内で書いた物がそのまま表示される感覚になります。
しかし、実際 JSX は JavaScript の拡張であるため、コンパイルする時は画面描画できる形に変換されます。
なので、ここでは実際にどう変換されるかを見ていきます。
そのためにまず React プロジェクトを作成します。
React プロジェクトを作成する方法は何でもいいですが、今回は[Vite](https://ja.vitejs.dev/)を使用しています。
Vite をインストールしたのち、`npm create vite@latest`で React のプロジェクトを作成したら、以下のコマンドでモジュールをインポートします。

```tsx
npm install --save-dev @babel/core @babel/preset-react
```

インポート後、vite.config.js(もしくは ts)を以下のように変更します。

```tsx
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
// https://vitejs.dev/config/
export default defineConfig({
  plugins: [
    react({
      jsxRuntime: "classic",
      babel: {
        plugins: [
          [
            "@babel/plugin-transform-react-jsx",
            {
              pragma: "React.createElement",
              pragmaFrag: "React.Fragment",
            },
          ],
        ],
      },
    }),
  ],
});
```

ここまでコンパイルの準備が完了したので、実際に実行します。
まず App.jsx(tsx)を以下のようにします。

```tsx
function App() {
  return <p>テスト</p>;
}
export default App;
```

その後、`npm run build`を実行します。
すると、dist/assets ディレクトリ配下に javascript ファイルが作成されます。
その js ファイルを開き、フォーマット後最後の方に以下のような記述があると思います。

```tsx
function wd() {
  return React.createElement("p", null, "テスト");
}
```

これが先程 JSX 記法で作成したコンポーネントを描画する処理になります。
React.createElement は[ドキュメント](https://ja.react.dev/reference/react/createElement)を確認すると以下の説明と、定義が存在します。

> `createElement` によって React 要素を作成できます。これは [JSX](https://ja.react.dev/learn/writing-markup-with-jsx) を書く代わりの手段として利用できます。

```tsx
const element = createElement(type, props, ...children);
```

どうやら、React において JSX 記法以外で画面描画を行うための記載方法みたいです。
コンパイル結果が React.createElement になることが確認できたので、引数についてみていきます。

### type

これは HTML のタグもしくは、React 独自のコンポーネント(例：Fragment)を指定します。

### props

コンポーネントに渡すオブジェクトになります。
オブジェクトなので、渡し方は例えば以下の通りです。

```jsx
function Greeting({ name }) {
  return createElement("h1", { className: "greeting" });
}
```

### children

作成するコンポーネントの子ノード部分になります。
ここに JSX での children と同様のものを設定します。
例えば、JSX で以下のように記載があったとします。

```jsx
function Greeting({ name }) {
  return (
    <h1 className="greeting">
      Hello <i>{name}</i>. Welcome!
    </h1>
  );
}
```

React.createElement を使用して書き直すと以下の通りです。

```jsx
function Greeting({ name }) {
  return createElement(
    "h1",
    { className: "greeting" },
    "Hello ",
    createElement("i", null, name),
    ". Welcome!"
  );
}
```

React.createElement の children 部分は`…children`となっているように、複数の引数を設定できます。
なので、描画させたい内容が複数ある場合は、表示させたい順番で第三引数以降に値をセットすれば良いです。
以上のようにコンパイル結果とその中で使用されているものについて、確認しました。
それでは、本題である以下の Test2 コンポーネントを使用した時、何も表示されない理由についてみていきます。

```jsx
function Test({ children }: { children: ReactNode }) {
  return <>{typeof children === "string" ? <p>{children}</p> : null}</>;
}
function Test2() {
  const text = "test";
  return <Test>value:{text}</Test>;
}
```

## typeof children が string にならない理由

ここからは typeof children === ‘string’が true にならなかった理由についてみていきます。
が、本題のコードを見る前にまず以下のコードのコンパイル結果を確認します。

```jsx
function Test2() {
  const text = "test";
  return <p>value:{text}</p>;
}
```

このコードをコンパイルすると以下のようになります。

```jsx
function wd() {
  return React.createElement("p", null, "value:", "test");
}
```

React.createElement の children 部分は JSX 内で直接記載している「value:」と、変数 text の値が別々で設定されています。
続いて以下のコードをコンパイルします。

```jsx
import { ReactNode } from "react";
function Test({ children }: { children: ReactNode }) {
  return <p>{children}</p>;
}
function Test2() {
  const text = "test";
  return <Test>value:{text}</Test>;
}
```

その時、画面描画を行う部分は以下のようになります。

```jsx
function wd({ children: e }) {
  return React.createElement("p", null, e);
}
function Sd() {
  return React.createElement(wd, null, "value:", "test");
}
```

Test コンポーネントの描画に該当する関数 wd が Test2 コンポーネントに該当する Sd 関数内で、React.createElement を使って呼び出されています。
そして、その時 children 部分は`"value:", "test"`とそれぞれ二つの引数が設定されています。
ここで、createElement の定義を再度確認します。

```jsx
createElement(type, props, ...children);
```

children 部分はスプレッド構文が使用されています。
スプレッド構文は該当する引数を配列として受け取ります。
例えば、以下のコードをみてください。

```jsx
const test = (...args) => {
  console.log(args);
};
test(1, 2, 3, 4);
```

この時、出力される値は`[1,2,3,4]`となります。
このことから、Test コンポーネントに渡される値は`[’value:’,’test’]`となることが分かります。
受け取る値が`[’value:’,’test’]`なので、typeof で型を判定しようとした時に string ではなく、object になります。
これが、最初示したコードで要素が表示されない理由になります。
`typeof children === 'string'`が true にならないのは処理を追うと自然だと感じますが、JSX で書いているが故に動きが分かりにくくなっていると思いました。
とはいえ、children の挙動について理解できたのは良かったなとおもっています。
なので、今後 React 内での typeof の使用と、children の解釈について気を付けて使用するようにします。

## 今回のようなケースでも文字列判定にしたい

先程までは children の中に、動的に設定される文字列とべた書きの文字列があるとき、
typeof の挙動が想定と異なる理由についてみていきました。
挙動は理解できたとしても、文字列しか実質設定していないのに、false になるケースを想定しながらの使用は個人的に難しいと感じます。
なので、ここではこれまで紹介している以下のコードのケースでも、true として判定できるようにします。

```jsx
function Test2() {
  const text = "test";
  return <Test>value:{text}</Test>;
}
```

そのために使用するのは、[Children](https://ja.react.dev/reference/react/Children)です。
この Children は props に含まれる children ではなく、children を操作するのに使用できる React の機能となっています。
今回はその中にある[Children.toArray](https://ja.react.dev/reference/react/Children#children-toarray)を使用します。
これは、受け取った children を配列にして格納します。
children は React.createElement の引数 children 部分に渡す値が 2 つ以上の場合は、配列で受け取りますが、一つしかない場合は配列ではないです。
そのため、どのような children の渡し方でも、配列として判定できるようにするために Children.toArray を使用し、children を配列とするようにします。
後は、配列になった children に対して全て文字列かを判定すれば、期待する動作をします。
具体的には以下のコードです。

```jsx
import { Children, ReactNode } from "react";
function Test({ children }: { children: ReactNode }) {
  const childrenElms = Children.toArray(children);
  const isOnlyStr = childrenElms.every((child) => typeof child === "string");
  return <>{isOnlyStr ? <p>{children}</p> : null}</>;
}
function Test2() {
  const text = "test";
  return <Test>value:{text}</Test>;
}
```

こうすることでは、先程は画面に何も表示されませんでしたが、今回は「value:test」が画面に描画されます。
children に渡すものが文字列であれば、どのような渡し方でも文字列として処理したい場合、[Children.toArray](https://ja.react.dev/reference/react/Children#children-toarray)を使用して判定することができます。
ただ、[ドキュメント](https://ja.react.dev/reference/react/Children)の先頭にも書いてあるように、Children の使用は推奨されていません。
そのため、可能な限り Children ではなく、そもそも判定方法を変えるなど別の手段に置き換えるほうが良さそうです。

## 余談 いつのまにか消えた import React from ‘react’のおまじない

少し前は React を使って実装する時、必ず`import React from ‘react’`をファイルに記載する必要がありました。
これを書かないと以下のようにエラーが発生するので、理由はわからないけどとりあえず書いていました。
![image.png](/images/react-compile-jsx/image.png)
この理由が今回コンパイル回りを調べてみて分かりました。
昔の React はコンパイルした結果、画面描画部分のコードは以下のように出力されます。

```jsx
function wd({ children: e, text: n }) {
  return React.createElement("p", null, e, n);
}
function Sd() {
  return React.createElement(wd, null, "value:", "test");
}
Za(document.getElementById("root")).render(
  React.createElement(Xi.StrictMode, null, React.createElement(Sd, null))
);
```

当然のように React オブジェクトを使用した処理になるようにしています。
これは、React オブジェクトをインポートしていようがいまいが関係なく生成されていました。
この生成されるコードのせいで、特におまじないを書かずに実行すると先程のエラーを発生させます。
なので、必ず使いもしない`import React from ‘react’`を書くことで、エラーの回避を行っていました。
しかし、いつしか`import React from ‘react’`のおまじないが無くても動くようになっていました。
その理由が[こちらの記事](https://zenn.dev/uhyo/articles/react17-new-jsx-transform)を参照すると、どうやら React17 からコンパイル結果が変わるようになったのが、要因みたいです。
元々は React.createElement を使っていたのが、jsx 関数による描画に変更されました。
その変更を確認するために、バージョン 18.3.1 の React で先程 React.createElement となっていた部分がどうなるかを確認します。
コンパイル結果は以下の通りです。

```jsx
function Ld({ children: e }) {
  return zr.jsx("p", { children: e });
}
function Td() {
  return zr.jsxs(Ld, { children: ["value:", "test"] });
}
ec(document.getElementById("root")).render(
  zr.jsx(Vu.StrictMode, { children: zr.jsx(Td, {}) })
);
```

jsx を用いて、描画されていますね。
このコンパイル結果の変更で、React オブジェクトを使用することが無くなり、`import React from ‘react’`のおまじないが無くても動くようになりました。
面白いですね。
なお、あくまで変わったのはコンパイル結果であり、React を使用する開発者が以前と書き方が大きく変わったということはないと思います。
なので、これまで見てきた children の扱いや対応についても同様のものとなっています。
また、今回余談より前で見てきたコンパイル結果を出力するために、`@babel/core @babel/preset-react`をインストールし、vite.config.ts の plugin を設定しています。
これは、以前のコンパイル結果を出力させたいがための設定なので、今 React で開発する人がこの設定をする必要は基本ないです。
むしろ、特別な事情がない限り設定はしない方がいいと思います。

## おわりに

今回は children で不思議だと思った挙動から、コンパイル結果を確認し、どういった対処方があるかを見てきました。
JSX は書き方 HTML ライクなので、基本開発する時は便利だと思っています。
ただ、JSX の分かりやすさに慣れてしまった故に、今回のような時どういった挙動をするかが分からず悩んでしまいました。
調べることで、ある程度解消できましたが、まだまだ React の理解が浅いなと実感しています。
引き続き勉強していけたらと思います。
ここまで読んでいただきありがとうございました。

## 参考資料

[https://ja.react.dev/reference/react/createElement](https://ja.react.dev/reference/react/createElement)
[https://zenn.dev/uhyo/articles/react17-new-jsx-transform#新しい jsx の変換結果](https://zenn.dev/uhyo/articles/react17-new-jsx-transform#%E6%96%B0%E3%81%97%E3%81%84jsx%E3%81%AE%E5%A4%89%E6%8F%9B%E7%B5%90%E6%9E%9C)
[https://ja.legacy.reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html](https://ja.legacy.reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html)
