---
title: "Viteのビルド時HTML要素に存在する指定のdata属性を削除する方法について"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vite", "React", "Vue"]
published: true
---

## はじめに

皆さんはフロントエンドのテストをするときに、要素を一意に判別するために何を使用していますか？
私は HTML にある data 属性を使用して、要素を取得できるようにします。
これによって、画面の動作に影響を与えることなく要素を取得できるようになります。
ただ、私の data 属性の使い方はテストのためだけに存在しているもので、ユーザーが使うためには全く不要な要素です。
そのため、リリースした時には`console.log`と同じように、存在しても大きな影響はないけど余計な情報を与える不要なコードとして残ってしまいます。
これは適切ではないと思い、画面に表示させるときには data 属性を表示させないようにする方法はないかを探しました。
探した結果見つけることができたので、今回はその方法を共有します。
なお、今回記載するのは Vue と React のみです。

## 先に結論

### Vue の場合

vite.config.ts を以下のようにします。

```tsx
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
// https://vitejs.dev/config/
export default defineConfig({
  plugins: [vue({
    template: {
      compilerOptions: {
        nodeTransforms: [
          (page) => {
            if (page.type === 1) {
              page.props = page.props.filter((prop) =>  prop.name.indexOf('data-test') === -1)
            }
          }
        ]
      }
    }
  })]
}
```

### React の場合

まず任意の Javascript ファイルを作成して、以下のコードを記載します。

```js
const removeCustomTestDataAttributes = () => {
  return {
    visitor: {
      JSXOpeningElement: (path) => {
        path.node.attributes = path.node.attributes.filter(
          (atr) => !atr.name.name.includes("data-test")
        );
      },
    },
  };
};
export default removeCustomTestDataAttributes;
```

そして、プロジェクトフォルダの配下に以下のコードを記載した.babelrc ファイルを作成します。

```.babelrc
{
    "plugins": [
        // パスとファイル名は適宜作った階層やファイル名へ変更します。
        "./babel.compile.js"
    ]
}
```

なお、今回示したコードは「data-test」から始まる data 属性のみを削除するコードとなっています。
適宜消したい data 属性が異なる場合は、置き換えて使用してください。

## ちょっとだけコードなどの解説

### Vue 編

Vite が用意している Vue プラグインには、単一コンポーネントファイル(SFC)の設定を行う template プロパティが用意されています。
なので、まず template プロパティを指定しています。
template プロパティには`compiler`、`compilerOptions`、`preprocessOptions`、`preprocessCustomRequire`、`transformAssetUrls`
各項目について、すごくざっくり説明します。
**compiler**
コンパイルやパーサーなどをゴリゴリにカスタマイズしたいときに使うとおもいます。
今回の目的ではこちらでも達成できたかもしれませんが、そこまでしたいことは多くないので、使ってません。
**compilerOptions**
自分でゴリゴリにコンパイルやパーサーなどの設定をしたいわけではないが、Vite が用意した範囲で良いので、何かしらの設定を加えたいときはこちらを使用します。
**preprocessOptions**
スクリプトコードを生成する時に指定するオプションです。
ただ、ソースコードを読んだ感じ@vitejs/plugin-vue では使用しておらず、[@vue/compiler-sfc](https://www.npmjs.com/package/@vue/compiler-sfc#vuecompiler-sfc)内の[render 関数](https://github.com/vuejs/core/blob/main/packages/compiler-sfc/src/compileTemplate.ts#L74)実行時に使いそうな感じがします。
これ以上は型が any であることから、追えなかったです。
なので、Vue の根本的な理解がある人用のオプションな気がします。
**preprocessCustomRequire**
これはプロジェクトの中以外の SFC を使用するときに、require のエラー解消を行うために使用します。
**transformAssetUrls**
画像などを画面で表示できるよう URL をよしなに生成してくれる機能の設定です。
デフォルトの設定はプロジェクトの URL ですが、特別な事情で参照する画像のルート URL を変更したいときに使用します。
今回のブログでは`compilerOptions`を使用しています。
`compilerOptions` にはコンパイルの時に Node の中身を設定できる`nodeTransforms`というオプションがあるので、そちらを使用します。
Node とはかなり乱暴に言ってしまえば、HTML 要素を JS 系で扱えるようにしたオブジェクト※だと思っていてください。
※この説明はかなり乱暴だと思うので、[こちらの記事](https://zenn.dev/sqer/articles/2d4def0f07bf04c5cc47)をはじめ、正確な情報の参照をお願いします。
`nodeTransforms` は引数に Node の情報を持つ関数の配列を指定でき、その中で Node の操作ができます。
なので、今回は以下のような処理を行っています。

```tsx
(page) => {
  if (page.type === 1) {
    page.props = page.props.filter(
      (prop) => prop.name.indexOf("data-test") === -1
    );
  }
};
```

まず、Node の中で各 HTML 要素を取得したかったので、type が 1 のもので絞り込んでいます。
その後、HTML の属性情報を持つ props プロパティを取得します。
ただ、今回は「data-test」から始まる属性は不要なので、「data-test」属性以外の属性だけを filter 関数で取得し直して、属性を上書きしています。
これによって、「data-test」属性のみビルド時にコードから削除されます。
実際にビルドをしてみて、「data-test」から始まる属性を付与したコードが存在していないことを確認してください。

### React 編

先程 Vue では template プロパティがあり、さらにその中にあるオプションで指定の属性を消すことができました。
では同様に React のプラグイも Node を操作できないかなと思い引数の型を確認したところ、以下の通りでした。

```tsx
interface Options {
  include?: string | RegExp | Array<string | RegExp>;
  exclude?: string | RegExp | Array<string | RegExp>;
  jsxImportSource?: string;
  jsxRuntime?: "classic" | "automatic";
  babel?:
    | BabelOptions
    | ((
        id: string,
        options: {
          ssr?: boolean;
        }
      ) => BabelOptions);
}
```

Node の取得が出来そうなのは、babel プロパティかなと思いさらに見てみました。

```tsx
export interface TransformOptions {
  auxiliaryCommentAfter?: string | null | undefined;
  auxiliaryCommentBefore?: string | null | undefined;
  rootMode?: "root" | "upward" | "upward-optional" | undefined;
  configFile?: string | boolean | null | undefined;
  babelrc?: boolean | null | undefined;
  babelrcRoots?: boolean | MatchPattern | MatchPattern[] | null | undefined;
  browserslistConfigFile?: boolean | null | undefined;
  browserslistEnv?: string | null | undefined;
  cloneInputAst?: boolean | null | undefined;
  envName?: string | undefined;
  exclude?: MatchPattern | MatchPattern[] | undefined;
  code?: boolean | null | undefined;
  comments?: boolean | null | undefined;
  compact?: boolean | "auto" | null | undefined;
  cwd?: string | null | undefined;
  caller?: TransformCaller | undefined;
  env?:
    | { [index: string]: TransformOptions | null | undefined }
    | null
    | undefined;
  extends?: string | null | undefined;
  filenameRelative?: string | null | undefined;
  generatorOpts?: GeneratorOptions | null | undefined;
  getModuleId?:
    | ((moduleName: string) => string | null | undefined)
    | null
    | undefined;
  highlightCode?: boolean | null | undefined;
  ignore?: MatchPattern[] | null | undefined;
  include?: MatchPattern | MatchPattern[] | undefined;
  minified?: boolean | null | undefined;
  moduleId?: string | null | undefined;
  moduleIds?: boolean | null | undefined;
  moduleRoot?: string | null | undefined;
  only?: MatchPattern[] | null | undefined;
  overrides?: TransformOptions[] | undefined;
  parserOpts?: ParserOptions | null | undefined;
  plugins?: PluginItem[] | null | undefined;
  presets?: PluginItem[] | null | undefined;
  retainLines?: boolean | null | undefined;
  shouldPrintComment?:
    | ((commentContents: string) => boolean)
    | null
    | undefined;
  sourceRoot?: string | null | undefined;
  sourceType?: "script" | "module" | "unambiguous" | null | undefined;
  test?: MatchPattern | MatchPattern[] | undefined;
  targets?:
    | string
    | string[]
    | {
        esmodules?: boolean;
        node?: Omit<string, "current"> | "current" | true;
        safari?: Omit<string, "tp"> | "tp";
        browsers?: string | string[];
        android?: string;
        chrome?: string;
        deno?: string;
        edge?: string;
        electron?: string;
        firefox?: string;
        ie?: string;
        ios?: string;
        opera?: string;
        rhino?: string;
        samsung?: string;
      };
  wrapPluginVisitorMethod?:
    | ((
        pluginAlias: string,
        visitorType: "enter" | "exit",
        callback: (path: NodePath, state: any) => void
      ) => (path: NodePath, state: any) => void)
    | null
    | undefined;
}
```

あまりいいのがない…。
`PluginItem`や`TransformCaller`の型も見ていきましたが、Node の操作が出来そうなものはありませんでした。
欲しいものがなく、途方にくれていましたが[Readme](https://github.com/vitejs/vite-plugin-react/tree/main/packages/plugin-react#babel)を読んでいたら以下の記載があり、ふと気づきました。

```tsx
// Use .babelrc files
babelrc: true,
```

「もしかして、自分で babel のプラグインを設定する必要がある？」と。
実際にそうでした。
なので、今回は.babelrc ファイルを作成して Babel で起動するプラグインを設定するようにしました。
実際の処理は Vue 側でやりたかったことと同じなので、詳細は省略しますが以下の点が注意です。
① 関数の戻り値は以下のように visitor プロパティを設定し、その中に JSXOpeningElement プロパティを設定しないと目的の操作はできません。

```js
{
 visitor: {
  JSXOpeningElement:(path)=> {
    path.node.attributes = path.node.attributes.filter((atr) => !atr.name.name.includes('data-test'))
   },
  },
 }
```

各プロパティで実行タイミングが違うそうなのですが、どのプロパティがどのタイミングかまでは把握できていないです。
②Javascript で書いているので、型補完が効かない。
これは私が Babel をあまり分かっていないことが原因です。
Typescript で書こうとしましたが、そもそも適切な型定義が分かりませんでした。
また、Typescript で作ったものを Babel に上手く読み込ませる方法もわからなかったので、今回は Typescript で作成するのは断念して、Javascript で作成しました。
以上で画面に表示させるときは、指定の data 属性を削除できるようになりました。
この処理と環境を検知できる変数を設定しておけば、テスト用の環境では E2E 試験用に data 属性を残しておく一方で、本番環境では data 属性は表示しないようにするといったことができます。
是非お試しください。
そして、ちゃんと全ての状況で動くか検証していないので、試して問題なさそうでしたら是非とも教えてください。

## おわりに

今回はコードをビルドした時に指定の data 属性を消去する方法について見ていきました。
終わって見れば、たかだか 10 数行で達成できたのですが、情報があまりに少なくて苦労しました。
特に辛かったのが、babel 周りです。
typescript で上手く動かせなかったし、ソースを読んでもちんぷんかんぷんでした。
なので、typescript で書くのは諦めて javascript にしてしまいましたが、とりあえず目的は達成出来て良かったです。
ここまで読んでいだきありがとうございました。

## 参考資料

[vue 側で参考とした投稿。これで、何をすればいいかの方向性が見えた](https://stackoverflow.com/questions/67922415/removing-all-data-test-attributes-from-vue-templates-during-production-build-in)
[React コンポーネントに設定された属性を取得する際に参考とした](https://stackoverflow.com/questions/61992884/using-babel-plugin-to-convert-jsx-attribute-into-a-function-call)
[もしかして React の場合、babel の話じゃね？と思うきっかけになった部分](https://github.com/vitejs/vite-plugin-react/tree/main/packages/plugin-react#babel)
