---
title: "Module Federation辞めてみた"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NextJS","ModuleFederation","MicroFrontends"]
published: true
---
## はじめに
[以前の記事](https://zenn.dev/maronn/articles/module-federation-nextjs)で、Next.jsを使ったModule Federationを構築した記事を書きました。
あれから、Module Federationを使って画面の構築を行っていました。
ただ、諸々の要因からModule Federationの使用を断念しました。
今回はそのような話です。
なお、先に結論を言っておくと断念した理由は以下の通りです。
- ビルドがとにかく重い。なんなら、Out of Memoryでエラーにすらなる
- ライブラリの依存解消が上手く行かなかった
- 一部型を生成できない

なお、上記理由はModule Federationに対する理解の浅さが原因の部分もあるかと思います。
なので、実はこうすれば解消できるよなどあれば、是非ともコメントいただけますと幸いです。
## 前提
断念した理由について話す前に、Module Federationをどう使っていたかについて記載します。
私が携わったアプリケーションでは、Module Federationをバックエンドの処理も内包するUIライブラリとして提供するために使用しました。
マイクロフロントエンドの文脈では、水平分割的な使い方をしていました。
https://note.com/dev_onecareer/n/nab16e5588c9e
垂直分割のような形で使っていなかったことはご留意ください。
また、Module Federationの構築の仕方は、[以前の記事](https://zenn.dev/maronn/articles/module-federation-nextjs)を元に行っているので、参照いただけますと幸いです。
## 断念した理由①：ビルドが重くなる
Module Federationは使用するJavascriptファイルをリモートで提供します。
それによって、そのファイルを使用するタイミングでのみ読み込むことができるので、ビルド量の削減に繋がります。
https://webpack.js.org/concepts/module-federation/
という触れ込みなので、ビルド量自体は軽くなりそうです。
しかし、上記を行うための設定をnext.config.jsに書くと相当重くなります。
コードで言うと、以下の部分です。
```yaml
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
```
ここ自体はリモートファイルを呼べるようする準備をしているだけの認識ですが、これがあるとビルドが相当遅くなり、next devの起動がまともにできないケースが多発します。
ギリギリnext devでアプリケーションを起動できても、Module Federationで提供するコンポーネントを使用した画面にアクセスすると、Out of Memoryで落ちます。
CIでもビルドを実行すると、メモリエラーで落ちることが多々ありました。
これに絡んだイシューも挙がっていますが、解消方法はメモリを増強するしかないという結論になっていそうです。
https://github.com/module-federation/core/issues/718
一応、next buildコマンドに関しては`NODE_OPTIONS=--max_old_space_size=4096 next build`とメモリ増強する形で実行すれば回避はできます。
しかし、next devコマンドでは変わらず重いままなので、ローカルでの開発体験は変わらず厳しいです。
このように、Module Federationを使用するとビルド周りが重くなり、開発体験が悪くなるのが理由として挙げられます。
この理由がModule Federationを断念した要因の8~9割ほどを占めます。
## 断念した理由②：ライブラリの依存解消が上手く行かなかった
Module Federationは元々の機能や、sharedオプションの指定なので、リモートファイルに使用されている依存パッケージの解消をよしなにやってくれる認識です。
実際、あるプロジェクトでは、Module Federationによるリモートファイルに使用されいてるパッケージを、アプリケーション内にインストールしなくても使用できました。
しかし、他のプロジェクトで同様の設定をして、アプリケーションを起動しようとしたら、依存エラーで実行できませんでした。
この問題がModule Federationを辞めるトドメになりました。
ただ、ビルドが重すぎる問題で疲弊した中だったため、原因はちゃんと調査できていないです。
## 断念した理由③：一部型を生成できない
以下のように[urql](https://commerce.nearform.com/open-source/urql/docs/)のProviderやReactのSuspenseで囲んだコンポーネントについては、エラーで上手く型が生成できませんでした。
```tsx
/** urql */
import { Client, Provider, cacheExchange, fetchExchange } from 'urql';
const client = new Client({
  url: 'http://localhost:3000/graphql',
  exchanges: [cacheExchange, fetchExchange],
});
export const App = () => (
  <Provider value={client}>
    <YourRoutes />
  </Provider>
);
/** ReactのSuspense */
import { Suspense } from 'react'
export const App = () => (
  <Suspense>
	<Hoge />
  </Suspense>
)
```
なので、Typescriptで開発しているプロジェクトで当該コンポーネントをインポートする際は、`@ts-expect-error`などで明示的にエラーを回避させる必要があります。
また、補完も効かなくなるので、propsを設定した場合はちゃんと値が適切かの責任を持つ必要があります。
このように、Typescriptを使っている場合、一部型補完が効かなくなります。
ただ、これはどちらかというとライブラリ起因な気もしますし、Suspenseについては使わないようにするなど回避策あるので、他二つの理由に比べるとそこまで断念には繋がりませんでした。
これがModule Federationの断念につながったのは、1~3%くらいの割合でしかないです。
0ではないので、一応紹介させていただきました。
## でも、置き換えは簡単だった
ここまでModule Federationを断念した理由を見てきました。
もう少しModule Federationと向き合えば解消したことはあったかもしれません。
しかし、Module Federationはお試し的な形で導入した手前、UIライブラリを使用するよりも負荷が大きく上がるなら、使い続けることはできません。
なので、Module Federationは断念し、UIライブラリを使う形で対応を変更しました。
この置き換えは非常に簡単でした。
すでに、コンポーネントという形で使用しているので、コンポーネントのimport先を書き換えるだけで動きました。
後は、Module Federation用の設定を削除するだけで、追加の設定はなかったです。
このように、UIライブラリと似た形での使用であれば、Module Federationを辞めることは容易なため、Module Federationを試してみるのはありだと思います。
## Module Federationを試してみる価値がある状況
理解も経験も浅いですが、Module Federationに触れてみた身として、Module Federationを試してみると良さそうな状況について感想を述べます。
それは、使用側・提供側ともにModule Federationをベースにした開発を行うことです。
今回のケースでは、以下のように一部のコンポーネントのみModule Federationで使用しました。
![ModuleFederation.drawio.png](/images/quit-module-federation/ModuleFederation.drawio.png)
しかし、これはアプリケーション内のビルドに加え、Module Federationの設定のビルドを行う必要がでてしまい、今回紹介した問題につながってしまいます。
なので、以下のように全てをModule Federationにすると、アプリケーション内のビルドは無くなるので、上手くいきそうです。
![ModuleFederationAll.drawio.png](/images/quit-module-federation/ModuleFederationAll.drawio.png)
実際には試していないので、本当に上手くいく保証はありません。
ただ、少なくともModule Federation本来の想定に沿った形で使用できてそうなので、今回のやり方よりは良い結果をもたらしそうです。
## おわりに
今回はModule Federationを断念した経緯などを書きました。
ビルドが耐えられないレベルで重くなるのは、正直想定外でしたが、使ってみないと分からないこともあるので、試せて良かったです。
またModule Federationは、部分的な導入ではなくプロジェクト全体導入していく必要があることも学べて良かったです。
Module Federation自体は魅力的な技術なので、もし今回記載したことの解消方法などご存知でしたら、是非とも教えてください！
ここまで読んでいただきありがとうございました。