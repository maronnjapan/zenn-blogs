---
title: "Reactを使いつつ、イベント伝播について改めて理解してみる"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "React"]
published: true
---

## はじめに

ある日の夕飯での話。
私：「今日ボタン押したら、その要素以外も反応してしまって困ったよ」
相手：「そうなんだ。なんで？」
私：「どうやらイベントの伝播を止めてなかったのが原因ぽいんだ」
相手：「ふーん、でどうやって解消したの？」
私：「stopPropagation したら、上手くいったんだ」
相手：「そっか、なんでそれで上手くいくの？」
私：「え、いや…、イベントの伝播を止めるのは stopPropagation だからだけど…。」
相手：「違う、違う。知りたいのはなんで stopPropagation をすることで、イベントの伝播を抑制すると言えるのかということ。そもそも、イベント周りの制御は preventDefault もあるでしょ？それとの違いは？後、イベントの伝播ってそもそも何？なんでそのボタンしかクリックしていないのに、他の要素にもイベントが伝わるってどういうこと？イベントが他の要素にも伝わるっておかしくない？そこら辺を説明してほしいの。」
私：「すみません、分かりません…。」
相手：「…」
私：「…」
この夕飯のくだりは創作ですが、実際に質問されたら答えられないなと思いました。
なので、今回は改めて Javascript でのイベントの伝播を中心に見ていこうと思います。
なお、コード部分は React を使用しています。
これは、私が現在 React を勉強中でそれを使用しているだけなので、React が説明しやすいというわけではないです。
よろしくお願いします。

## DOM のイベント伝播について

画面上にあるボタンをクリック際に発生クリックイベントなどはそのボタンそのものだけがイベント処理を実行するわけではありません。
下記の図のように、イベントは最上位の Window オブジェクトからイベントが順にイベントが伝播され、イベントの発生元となった要素まで到達したら、Window オブジェクトに向けてイベントが再度伝播していきます。
![**[JavaScriptとHTMLとDOMの基本#2 イベント編](https://gihyo.jp/dev/serial/01/crossbrowser-javascript/0007)より引用**](/images/js-event-handler/2023-12-16_10h47_01.png)
**[JavaScript と HTML と DOM の基本#2 イベント編](https://gihyo.jp/dev/serial/01/crossbrowser-javascript/0007)より引用**
そして、イベントの発生元から Window オブジェクトに向けてイベントが伝播していく、バブリングフェーズでクリックイベントに対しての処理が定義されていると、その処理が実行されます。
そのため、バブリングフェーズを経るときに、イベントを発生する元となった要素以外にクリックイベントが設定されているとその処理まで実行されてしまいます。
これは予期せぬ挙動を発生させる要因となるため、Javascript にはこの伝播を制御する機能が備わっています。

## React でイベントの伝播などの操作を行う

React の場合は、主に以下のようにしてイベントの制御を行います。

```tsx
const test = (e: React.MouseEvent) => {
  e.stopPropagation();
  e.preventDefault();
};
return (
  <>
    <button onClick={test}></button>
  </>
);
```

`e.stopPropagation()`は先程の図で見た、バブリングフェーズを発生させないものとなっています。
そのため、button 要素で発生したイベントが親に伝わって行かないため、実行されるイベントに対しての処理は button 要素のみとなります。
`e.preventDefault()`はイベントに備わっている動作をキャンセルするものです。
例えば、リンクをクリックしたらリンク先に遷移することや、submit ボタンをクリックするとフォームの内容を送信するといった、明示的に処理を実装しなくても動作するものをキャンセルします。
ちなみに、`e.preventDefault()`はイベントの伝播を抑制することはないので注意してください。
例えば下記のコードをみてください。

```tsx
function App() {
  const handleParent = () => alert("親");
  const handleMy = () => alert("私");
  const handleChild = (e: React.MouseEvent) => {
    e.preventDefault();
    alert("子供");
  };
  const parentStyle = {
    height: "200px",
    width: "200px",
    border: "1px solid black",
  };
  const myStyle = {
    height: "100px",
    width: "100px",
    border: "1px solid red",
    margin: "auto",
  };
  const childStyle = {
    display: "block",
    height: "50px",
    width: "50px",
    border: "1px solid black",
    margin: "auto",
  };
  return (
    <>
      <div onClick={handleParent} style={parentStyle}>
        親
        <div onClick={handleMy} style={myStyle}>
          現在要素
          <a href="https://google.com" onClick={handleChild} style={childStyle}>
            子供
          </a>
        </div>
      </div>
    </>
  );
}
export default App;
```

これを画面上に出すと以下のようになります。
![2023-12-16_11h10_51.png](/images/js-event-handler/2023-12-16_11h10_51.png)
この時、「子供」の枠内をクリックすると`e.preventDefault()`が設定されているので、リンク先には遷移しません。
しかし、イベントの伝播は行われるので、それぞれ順に「子供」、「私」、「親」とアラートが表示されます。
アラートも「子供」だけ表示するようにして、リンク先に遷移させないようにするには以下のように伝播の抑制とイベントのキャンセル両方を記載する必要があります。

```tsx
const handleChild = (e: React.MouseEvent) => {
  e.preventDefault();
  e.stopPropagation();
  alert("子供");
};
```

## React で prevnetDefault が上手く効かない場合について

`e.preventDefault()`はそのままでは効かないケースがあります。
それは Passive モードが有効になっているイベントに対してです。
例えば wheel イベントについて見ていきます。
以下のようなコードを書いた時に、感覚的としてはホイールを回してもスクロールしないようになると思います。

```tsx
function App() {
  const wheelParent = (e: React.WheelEvent) => e.preventDefault();
  const parentStyle = {
    height: "2000px",
    width: "200px",
    border: "1px solid black",
  };
  return (
    <>
      <div style={parentStyle} onWheel={wheelParent}>
        親
      </div>
    </>
  );
}
export default App;
```

しかし、実際はホイールを回すとエラーが発生します。
これはイベントにおける Passive モードが関係しています。
Passive モードとは、preventDefault が起きないと決めつけている状態となっています。
通常のイベントの場合、preventDefault が呼び出される可能性があるため、preventDefault が呼び出されていないかを確認してから、イベントの処理を行います。
ただ、この判定が入ることによって、ホイールイベントなどは実際の操作と画面への反映にずれが生じます。
そのため、React では wheel イベントや touch イベントなど即時反映が求められるイベントに対して、Passive モードを有効にしています。
Passive モードが有効になっているので、preventDefault でイベントをキャンセルしようとするとエラーを起こすことになります。
もし、どうしても Passive モードが有効になっているイベントの Passive モードを無効にしたい場合は以下のように記載します。

```tsx
function App() {
  const wheelParent = (e: React.WheelEvent) => e.preventDefault();
  const divRef = useRef<HTMLElement>(null);
  useEffect(() => {
    const div = divRef.current;
    div?.addEventListener(
      "wheel",
      (e) => {
        const event = e as unknown as React.WheelEvent<HTMLElement>;
        wheelParent(event);
      },
      { passive: false }
    );
    return () => {
      div?.removeEventListener("wheel", (e) => {
        const event = e as unknown as React.WheelEvent<HTMLElement>;
        wheelParent(event);
      });
    };
  });
  const parentStyle = {
    height: "2000px",
    width: "200px",
    border: "1px solid black",
  };
  return (
    <>
      <div style={parentStyle} ref="divRef">
        親
      </div>
    </>
  );
}
export default App;
```

これで Passive モードを無効にしつつ、wheel イベントの処理を実行することができます。

## React でイベントハンドラーをキャプチャリングフェーズで実行させる

これまでイベントの処理が走るのはバブリングフェーズと言いましたが、キャプチャリングフェーズでもイベントを実行することはできます。
設定したイベント属性の後ろに「Capture」を設定すれば、キャプチャリングフェーズの時にイベントに対しての処理を実行できます。
それを示すために以下のコードをみてください。

```tsx
function App() {
  const handleParent = () => alert("親");
  const handleMy = () => alert("私");
  const handleChild = (e: React.MouseEvent) => {
    e.preventDefault();
    alert("子供");
  };
  //...略
  return (
    <>
      <div onClickCapture={handleParent} style={parentStyle}>
        親
        <div onClick={handleMy} style={myStyle}>
          現在要素
          <a
            href="https://google.com"
            onClickCapture={handleChild}
            style={childStyle}
          >
            子供
          </a>
        </div>
      </div>
    </>
  );
}
export default App;
```

「親」を設定した div 要素に onClickCapture 属性を指定しています。
そして、「子供」のリンクをクリックした際、まず「親」のアラートが表示されてから、その後に「子供」、「私」の順番でアラートが表示されます。
このように、バブリングフェーズだけでなくキャプチャリングフェーズでもイベントを発生させることで、処理の順番を制御することができます。

## 余談 React で引数にイベントを渡すときの型について

現状確認した範囲での React のイベント型は以下のものがあります。

```tsx
interface ClipboardEvent
interface CompositionEvent
interface DragEvent
interface PointerEvent
interface FocusEvent
interface FormEvent
interface InvalidEvent
interface ChangeEvent
interface KeyboardEvent
interface MouseEvent
interface TouchEvent
interface UIEvent
interface WheelEvent
interface AnimationEvent
interface TransitionEvent
```

イベントを引数にする場合、Typescript だと any 型になってしまうので、明示的に型を指定する必要があります。
その際に、ご活用ください。
なお、値を変更するときに何かしらそれに関わる値を取得したいときは大体 ChangeEvent でまかなうことができそうでした。

## おわりに

今回は改めてイベントの伝わり方について見ていきました。
なんとなく分かっているという状態から、`stopPropagation`と`preventDefault`の意味や違い、そもそもイベントはどう発火するのかといったことがある程度説明できるようになりました。
次からは、`stopPropagation`や`preventDefault`を使用している意図が説明できそうなので良かったです。
ここまで読んでいただきありがとうございました。
