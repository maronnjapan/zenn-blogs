---
title: "Module FederationをNext.jsでやってみた"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ModuleFederation", "Next", "Javascript", "webpack"]
published: true
---

# はじめに

アプリケーションを一個の大きなプロジェクトで開発するのではなく、それぞれドメインごとに切り分け、各々が独立して動くようにするマイクロサービスが様々な場所で導入されています。
主観で恐縮ですが、マイクロサービスはバックエンドを各ドメインで分割している場面が多い気がします。
ただ、それぞれのドメインで分割したいのはバックエンドだけではありません。
フロントエンドでもドメイン単位で分割し、それぞれ独立して開発したいケースはあると思います。
そういった時に使えるのがマイクロフロントエンドという考えです。
そこで、今回はマイクロフロントエンドの一手法である Module Federation を使い、マイクロフロントエンドの一端をほんの少しだけ感じてみよう思います。
なお、この記事ではマイクロサービスやマイクロフロントエンドの概要など詳細は記載しません。
マイクロフロントエンドの手法の一つである Module Federation を中心に解説します。
では始めます。

# Module Federation とは

Module Federation とは Javascript のコードを小さな単位に分割したもの（≒ チャンク）を提供し、それを非同期・同期でロードをすることです。
![1677586047168.png](/images/module-federation-nextjs/1677586047168.png)  
[Building MicroFrontends with webpack Module Federation](https://www.linkedin.com/pulse/building-microfrontends-webpack-module-federation-gaurav-sharma/) より引用
コードをプロジェクト内にインストールするのではなく、リモートにあるコードを読むこむ形なので、必要な時に必要な形で画面を構成できます。
そして、リモートで読むことからローカルでの挙動とオンラインでの挙動に大きな違いはありません。
さらに Module Federation は共有するプロジェクト間でライブラリのバージョンを指定できます。
これによって、ライブラリを使用するプロジェクト用に特定のバージョンだけをロードします。
実際に検証していないので、断言はできませんが恐らく Module Federatio の shared でライブラリのバージョンを固定すれば、プロジェクト間で使用している同じライブラリのバージョンが大きく異なっていても動くのではないかと考えています。
チャンクで静的ファイルを配布し、ライブラリのバージョン違いも吸収できるなど、柔軟性は非常にありますが、デメリットにもなります。
自由度が高いので、プロジェクト間の共有の流れが双方向になってしまう場合があります。
そうなると、各プロジェクトが別プロジェクトに依存してしまうという分散モノリスに陥ってしまいます。
また、静的ファイルをリモートで配布するという制約上、必ずログインが必須となるプロジェクトでは Module Federation 用のエンドポイントは穴を開けておくなどの対応が必要です。
以上のように、便利な面は相当ありつつも、きちんと管理しないと巨大な分散モノリスになってしまうという懸念もあります。
この点を注意して使っていく必要はありますが、とりあえず動かして見ないと感じにくい部分もありますので、次からは実際に Next.js で Module Federation を試してみます。

# Next.js で Module Federation を搭載する

## モジュールの提供側

### 環境構築

今回は webpack を使用して Module Federation を行うので、`npx create-next-app@latest`といった crate-next-app コマンドでプロジェクトを作成します。
基本的に選択はオプションで良いと思いますが、App Router ではなく Pages Router を**必ず**選んでください。
プロジェクトを作成したら、[@module-federation/nextjs-mf](https://www.npmjs.com/package/@module-federation/nextjs-mf)をインストールします。
これで大まかな準備ができました。

### モジュールを export する

次にエクスポートしたいコンポーネントを作成します。
なお、今回はコンポーネントをエクスポートしますが、pages ファイルもエクスポートできます。
この柔軟性が Module Federation の強みなので気になる方は試してみてください。
src/components/Test.tsx ファイルを作成し、以下のコードを記載します。

```tsx
import { CSSProperties, ReactNode } from "react";
export interface TestProps extends CSSProperties {
  label: string;
  children: ReactNode;
}
function Test({ children, label }: TestProps) {
  return (
    <div>
      <p>{label}</p>
      {children}
    </div>
  );
}
export default Test;
```

ポイントとしては default を使用してコンポーネントを export している点です。
モジュールを使用する側でのインポートが簡単なので、特別な事情が無ければ default を使用した方が良さそうです。
これで Module Federation を提供側の準備が整いました。
実際にモジュールを export しましょう。

### next.config.js で Module Federation の設定をする

プロジェクト直下に存在する next.config.js を以下のように記載します。

```tsx
/** @type {import('next').NextConfig} */
const { NextFederationPlugin } = require("@module-federation/nextjs-mf");
const moduleConfig = {
  name: "next2",
  filename: "static/chunks/remoteEntry.js",
  exposes: {
    "./test": "./src/components/Test.tsx",
  },
};
module.exports = {
  webpack(config, options) {
    config.plugins.push(new NextFederationPlugin(moduleConfig));
    return config;
  },
};
```

ポイントは moduleConfig 変数です。
ここで、以下のプロパティを設定します。
① モジュール全体を示す名前：name
②export するモジュールの処理が記載してあるファイルの名前：filename
③ 実際に export するモジュールの名前とファイルのパス
後はこのオプションを先程インストールした@module-federation/nextjs-mf 内にある NextFederationPlugin へ食わせてあげます。
これでビルドすれば Module がまとまったファイルを提供できます。
実際にやってみましょう。
package.json の scripts にある dev スクリプトをを以下のように変更し、実行します。

```tsx
"dev": "PORT=3001 NEXT_PRIVATE_LOCAL_WEBPACK=true next dev”
```

PORT はこの後使用する側のことも考慮して変えているだけで必須の設定ではありません。
ただし、`NEXT_PRIVATE_LOCAL_WEBPACK=true`部分は設定しないとビルドが完了できないので、必ず記載してください。
起動したら、[http://localhost:3001/\_next/static/chunks/remoteEntry.js](https://test-module-federation-export-component.vercel.app/_next/static/chunks/remoteEntry.js)にアクセスしてください。
すると、以下のようなファイルが表示されます。

```tsx
!function(){var __webpack_modules__={6478:function(e,t,n){"use strict";n.r(t);var r=n(4870),o=n.n(r),i=n(4347),a=n.federation;for(var s in n.federation={},o())n.federation[s]=o()[s];for(var s in a)n.federation[s]=a[s];if(!n.federation.instance){let e=[!!i.A&&(0,i.A)()].filter(Boolean);n.federation.initOptions.plugins=n.federation.initOptions.plugins?n.federation.initOptions.plugins.concat(e):e,n.federation.instance=n.federation.runtime.init(n.federation.initOptions),n.federation.attachShareScopeMap&&n.federation.attachShareScopeMap(n),n.federation.installInitialConsumes&&n.federation.installInitialConsumes()}},2013:function(e,t,n){"use strict";function r(){return"next2:0.1.0"}function o(){return!1}function i(){return"undefined"!=typeof window}n.r(t),n.d(t,{FederationHost:function(){return tI},getRemoteEntry:function(){return e7},getRemoteInfo:function(){return e5},init:function(){return tP},loadRemote:function(){return tN},loadScript:function(){return eZ.k0},loadScriptNode:function(){return eZ.oe},loadShare:function(){return tA},loadShareSync:function(){return tk},preloadRemote:function(){return tx},registerGlobalPlugins:function(){return L},registerPlugins:function(){return tD},registerRemotes:function(){return tj}});let a="[ Federation Runtime ]";function s(e,t){e||l(t)}function l(e){if(e instanceof Error)throw e.message=`${a}: ${e.message}`,e;throw Error(`${a}: ${e}`)}function u(e){e instanceof Error?(e.message=`${a}: ${e.message}`,console.warn(e)):console.warn(`${a}: ${e}`)}function c(e,t){return -1===e.findIndex(e=>e===t)&&e.push(t),e}function f(e){return"version"in e&&e.version?`${e.name}:${e.version}`:"entry"in e&&e.entry?`${e.name}:${e.entry}`:`${e.name}`}function h(e){return void 0!==e.entry}function d(e)
/** 省略 */
```

以上で提供は完了です。
ではこのモジュールを使用してみましょう。

## モジュールの使用

### 環境構築

モジュールの提供側と同じように Next.js のプロジェクトを作成し、[@module-federation/nextjs-mf](https://www.npmjs.com/package/@module-federation/nextjs-mf)をインストールします。

### next.config.js でインポートする

next.config.js に以下のような記載をします。

```tsx
/** @type {import('next').NextConfig} */
// next.config.js
const { NextFederationPlugin } = require("@module-federation/nextjs-mf");
module.exports = {
  webpack(config, options) {
    const moduleConfig = {
      name: "next1",
      remotes: {
        next2: `next2@http://localhost:3001/_next/static/chunks/remoteEntry.js`,
      },
      filename: "static/chunks/remoteEntry.js",
    };
    config.plugins.push(new NextFederationPlugin(moduleConfig));
    return config;
  },
};
```

基本的にモジュールの提供側と同じですが、モジュールの提供側では expose だったのが、使用する側では remotes プロパティを設定しています。
この remotes には、モジュールの提供側で定義した名前と remoteEntry.js にアクセスできる URL を@で繋げます。
これでインポートの準備が完了です。

### コンポーネントを使用する

それでは早速使用してみます。
任意のファイルで以下のように記載します。

```tsx
import dynamic from "next/dynamic";
const Test = dynamic(() => import("next2/test"), { ssr: false });
export default function Home() {
  return (
    <Test label="eeee">
      <p>これはモジュールをインポートしていますuuuuuuuuuu</p>
    </Test>
  );
}
```

今回は静的なコンポーネントなので、dynamic を使用する必要はないのです。
ですが、提供するコンポーネントが外部リクエストなどで動的に変わる際は遅延ロードをしないと Hydration Error が発生します。
なので、基本的には遅延ロードしておいた方が安牌です。
なお、[ドキュメント](https://github.com/module-federation/core/tree/main/packages/nextjs-mf#usage)では React.lazy の使用を推奨していますが、dynamic でも Hydration Error は起きなかったので、実験の意味も兼ねて dynamic を使用しています。
package.json に`"dev": "PORT=3001 NEXT_PRIVATE_LOCAL_WEBPACK=true next dev”`スクリプトを追加し、実行した後 http://localhost:3000 にアクセスすると以下のような画面が表示されます。
![2024-07-23_23h21_13.png](/images/module-federation-nextjs/2024-07-23_23h21_13.png)
おお！モジュール提供側で実装したコンポーネントが表示されていますね。
しかも、children や labelProps も効いていそうです。
このようにライブラリとして提供・インポートしなくても、他プロジェクトの実装を使用できます。
柔軟性が Module Federation の強みと言いましたが、この簡単な例でもその可能性を感じますね。
破壊的な変更をしたときでもディレクトリを分け、バージョン 2 として提供することで以前の処理と新しい処理を合わせて使用できるなど妄想が膨らみます。
簡単な実装であれば非常に簡単なので、是非やってみてください。

## モジュールに型を付与する

先程は Module Federation の簡単な例を示し、アプリケーションが動くことを確認しました。
もちろん、あれでちゃんと動くのですが実は問題が残っています。
それは補完が一切効かないことです。
例えば、先程の例を VSCode で実装してみると以下のようなエラーが発生します。
![2024-07-23_23h29_28.png](/images/module-federation-nextjs/2024-07-23_23h29_28.png)
アプリケーションを動かすときは提供先 URL から対象のコンポーネントを取得するので、この実装でも動きます。
しかし、VSCode 上ではそんなこと知ったことではなく、node_modules に対象のモジュールがないので、エラーが発生します。
さらに Test コンポーネントも、そもそもインポートが VSCode 上ではできていないことになっているので、props の補完が効きません。
手入力で入れて上げればその値は反映されるのですが、何とも不便です。
よって、ここでは Module Federation を使用しても型補完が効くようにしていきます。

### 下準備

まずは、[@module-federation/typescript](https://github.com/module-federation/core/tree/main/packages/typescript)をそれぞれのプロジェクトにインストールします。
その後、それぞれの next.config.js に以下のような記載をします。

```tsx
/** モジュール提供側 */
const { NextFederationPlugin } = require("@module-federation/nextjs-mf");
const { FederatedTypesPlugin } = require("@module-federation/typescript");
const moduleConfig = {
  name: "next2",
  filename: "static/chunks/remoteEntry.js",
  exposes: {
    "./test": "./src/components/Test.tsx",
  },
};
module.exports = {
  webpack(config, options) {
    config.plugins.push(new NextFederationPlugin(moduleConfig));
    config.plugins.push(
      new FederatedTypesPlugin({
        federationConfig: moduleConfig,
      })
    );
    return config;
  },
};
/** モジュール使用側 */
const { NextFederationPlugin } = require("@module-federation/nextjs-mf");
const { FederatedTypesPlugin } = require("@module-federation/typescript");
module.exports = {
  webpack(config, options) {
    const moduleConfig = {
      name: "next1",
      remotes: {
        next2: `next2@http://localhost:3001/_next/static/chunks/remoteEntry.js`,
      },
      filename: "static/chunks/remoteEntry.js",
    };
    config.plugins.push(new NextFederationPlugin(moduleConfig));
    config.plugins.push(
      new FederatedTypesPlugin({
        federationConfig: moduleConfig,
      })
    );
    return config;
  },
};
```

これだけです。
後はそれぞれ dev スクリプトで実行し直すと、モジュールを使用するプロジェクト配下に@mf-types ディレクトリが作成され、型定義ファイルが生成されます。
これで補完が効くようになりました。
使用側で以下のようにインポートすると、

```tsx
const Test = dynamic(() => import("../../@mf-types/next2/test"), {
  ssr: false,
});
```

VSCode 上でも補完が効きます。
![2024-07-24_00h27_57.png](/images/module-federation-nextjs/2024-07-24_00h27_57.png)
ただし、このままでは動きません。
@mf-types ディレクトリはあくまで型定義ファイルしか存在しないので、そのパスでインポートしても実体がなくエラーが発生します。
動かすためには`next2/test`でインポートする必要があります。
これを解消するために、tsconfig.json に以下の設定を追加します。

```tsx
{
  "compilerOptions": {
    /** 省略 */
    "baseUrl": ".",
    "paths": {
      "*": ["./@mf-types/*"]
    }
  }
  /** 省略 */
}
```

上記のように設定することで、`next2/test`で VSCode 上では型定義ファイルを参照でき、アプリケーションを動かす時は実体が存在する`next2/test`を参照するということができます。
便利ですね。
ただ、この型定義生成の設定は場合によっては上手く生成されない可能性があります。
ハッキリと理解ができていないので、根本原因を説明できませんが以下のケースは上手く型定義ファイルが生成できませんでした。

- Next.js にある dynamic を使用したラッパーコンポーネントが最上位にあるコンポーネント
- urql の Provider が最上位にあるコンポーネント

便利は便利ですが、それなりに制約はありそうです。

# Next.js で Module Federation を行う際の課題

先程までは Moduel Federation を使った実装を行っていました。
便利で目新しい部分もあるので、夢が広がりますがもちろん課題もあります。
ここではそれについての話をします。
なお、Next.js というよりは今回使用したライブラリ起因の話がほとんどですが、ここでは Module Federation 実装する際に感じた課題を記載していきます。

## ビルドツールの制限

Module Federation を公式でサポートとしているのは、確認している範囲では[Webpack](https://webpack.js.org/concepts/module-federation/)と[Nx](https://nx.dev/recipes/module-federation)です。
ただ、Webpack は[こちらの記事](https://www.publickey1.jp/blog/23/javascriptwebpackturbopack.html)であるように、新規開発がストップすることが方向されています。
また近年は以下のように Vite の勢いがすさまじく、数年後に Vite での構築が当たり前になっている可能性は大いにあります。
![[State of Javascript 2023](https://2023.stateofjs.com/en-US/libraries/build_tools/) より引用](/images/module-federation-nextjs/2024-07-15_13h26_51.png)
[State of Javascript 2023](https://2023.stateofjs.com/en-US/libraries/build_tools/) より引用
依然として Webpack が一番となっていますが、使用されなくなってくる可能性を考えると、将来的な入れ替えコストは考える必要があります。
Nx は使ったことがなく、よくわからないのでここでは触れません。
なお、Vite は公式で Module Federation サポートしていませんが、[プラグイン](https://github.com/originjs/vite-plugin-federation)自体は開発されています。
そのため、将来的にはビルドツールの懸念はなくなるかもしれませんが、先のことは分からないので、ビルドツールに制限があることは Module Federation の課題として残り続けると思います。

## App Router では使えない

昨今 Next.js は App Router を打ち出し、App Router の使用を推奨しています。
なので、これから Next.js を使用する場合、App Router を用いることが多いと思います。
しかし、[今回使用したライブラリ](https://github.com/module-federation/core/tree/main/packages/nextjs-mf)は App Router に対応していません。
App Router で使えないかと、以下のようにイシューなどが立ち上がっていますが、ことごとくサポートしないという回答か、App Router の対応は出来ないとなっていました。
[https://github.com/module-federation/core/issues/1943](https://github.com/module-federation/core/issues/1943)
[https://github.com/module-federation/core/issues/1946](https://github.com/module-federation/core/issues/1946)
[https://github.com/module-federation/core/pull/2002](https://github.com/module-federation/core/pull/2002)
プルリクエストの最後には以下のようなコメントがモジュールのメンバーからありました。

> There will never be module federation with app router, not my ecosystem - not for next.js
>
> Vercel will make proprietary system for app router most likely.

「App Router で Module Federation は無理！Vercel が独自のものを作ってくれるやろう」、と回答者の App Router への苦悩を感じますね。
このように現在 Next.js で推奨されている App Router では使用できないことは、留意する必要があります。

## CORS の注意

Module Federation を行うと、コンポーネントなどをインストールせずに使用できます。
そして、使用するコンポーネントも動いているプロジェクトから引っ張ってきているので、外部リクエストも実行してくれます。
ただし、外部リクエストの URL には注意が必要です。
確かに、コンポーネントは元々コンポーネント提供側で動いているものを使用しています。
とはいえ、実際に使用したら、参照する URL は使用する側のものとなります。
具体例をみていきます。
例えば、以下のコンポーネントがあるとします。

```tsx
import { ReactNode, useEffect, useState } from "react";
export interface TestProps {
  children: ReactNode;
}
export function Test({ children }: TestProps) {
  const [user, setUser] = useState("");
  useEffect(() => {
    const fetchUsers = async () => {
      const response = await fetch("/api/user");
      const data = await response.json();
      setUser(data.name);
    };
    fetchUsers();
  }, []);
  return (
    <div>
      <p>{user}</p>
      {children}
    </div>
  );
}
export default Test;
```

コンポーネントのマウント時に、`/api/user`にリクエストを実行します。
このコンポーネントを実装しているプロジェクトには`/api/user`の API が存在し、プロジェクトの URL は http://localhost:3001 とします。
次に、Module Federation を用いて先程のコンポーネントを取得したとします。
そして、コンポーネントを提供側ではなく、取得側のプロジェクトで呼び出します。
取得側のプロジェクト URL は http://localhost:3000 とします。
この時、コンポーネントはマウント時 http://localhost:3001 ではなく、http://localhost:3000 にリクエストします。
ただ、取得側のプロジェクトには`/api/user`のエンドポイントはないので 404 エラーとなります。
上記を回避するために、コンポーネント内の fetch 関数はパスでなく、提供側のプロジェクト URL を指定する必要があります。
URL を設定することで、本来飛ばしたいエンドポイントにリクエストを投げることができます。
しかし、動いている環境の URL とは異なるので、そのままでは CORS エラーが発生します。
そのため、CORS の条件を緩和するなどの調整が必要となるのは、注意が必要です。

## スタイル被りによるレイアウト崩れ

Module Federation を用いて、別のコンポーネントなどを使用する場合、Javascript 的な機能以外にも CSS や HTML 周りの設定も取得します。
なので、各プロジェクトで prefix 的なものをつけ、レイアウトの衝突を避ける必要があります。

## Hydration は Module Federation でも注意が必要

正直 Module Federation の話でもなんでもないですが、Module Federatio を使用したとしても Hydration Error は相変わらず発生します。
なので、dynamic や lazy 関数を使用することで、遅延ロードなどの対応することは必要となります。

# おわりに

今回は Module Federation を Next.js で行ってみました。
結構簡単なのと、柔軟なエクスポート・インポートができるので、各ページをそれぞれ独立して開発し、それらを埋め込むことで一つのアプリケーションができそうなど、妄想が広がりました。
ただ、Moduele Federation は銀の弾丸ではないので、使って間もないですが制約はちょいちょい感じています。
そのため、Module Federation に固執することなく、とはいえ可能性を探っていくというスタンスで今後も理解を深めて行きたいです。
ここまで読んでいただきありがとうございました。
