---
title: "SCSSファイルをCSSに変換し、型補完を利かせる"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CSS", "SCSS"]
published: true
---

## はじめに

[styled-componets](https://styled-components.com/)をはじめとした CSS in JS、[Tailwind CSS](https://tailwindcss.com/)といった CSS フレームワーク。
これらの台頭により、CSS ファイルを作成し、その中に定義を書くことはどんどん少なくってきました。
しかし、時に CSS ファイルを作成しその中に定義を書く必要、書きたい事態は発生します。
その時に「型補完が効かなくて不便」と感じる人もいるのではないでしょうか？
私は実際にそう感じました。
なので、今回は CSS ファイルから型ファイルを生成し、型安全に使用できる方法について見ていきます。
なお、CSS ファイルと言っている中恐縮ですが、今回の方法は[CSS Modules](https://github.com/css-modules/css-modules)を使用することを想定しています。
通常の CSS の書き方でも同様に動作するかは検証していないので、その点はご容赦ください。

## CSS Modules から型ファイルを作成する方法

CSS Module から型定義ファイルを生成するために[typed-css-modules](https://github.com/Quramy/typed-css-modules)をインポートします。
typed-css-modules の内部の挙動については[こちらの記事](https://zenn.dev/cybozu_frontend/articles/2528ad2935be9f)が詳しいです。
ここでは使い方だけ見ていきます。
まず、任意の CSS Module ファイルを作成します。

```css
/* Test.module.css */
.test {
  font-size: large;
}
.test-part2 {
  font-size: larger;
}
.test .test-part3 {
  font-size: medium;
}
.test-part2 > .test-part4 {
  font-size: small;
}
```

そして、`npx tcm src`をターミナルで実行します。
すると以下のような型定義ファイルが生成されます。

```tsx
// Test.module.d.ts
declare const styles: {
  readonly test: string;
  readonly "test-part2": string;
  readonly "test-part3": string;
  readonly "test-part4": string;
};
export = styles;
```

後はこの`styles`を使用したいファイルにインポートすれば、クラス名の補完が出ます。
![2024-05-07_12h09_20.png](/images/css-scss-watch/2024-05-07_12h09_20.png)
設定すれば CSS Module ファイルで定義したクラス名をもとにした class 属性が付与されます。
便利ですね。
先に行く前に先程実行した tcm コマンドについて軽く見ていきます。
tcm コマンドは大枠として、`tcm <input directory>`という形式を取るコマンドです。
input directory 部分は CSS Module が含まれているディレクトリを指定します。
今回は src ディレクトリ配下にある CSS Module ファイルを対象にしたかったので、src を指定しています。
もし対象のファイルを指定したい場合は、[p オプション](https://github.com/Quramy/typed-css-modules?tab=readme-ov-file#file-name-pattern)があるのでそれを使用します。
そして、特にオプションを指定しなければ生成される型ファイルは CSS Module ファイルと同じ階層となります。
型ファイルの出力をしたい場合は[o オプション](https://github.com/Quramy/typed-css-modules?tab=readme-ov-file#output-directory)を指定します。
また、今は一回型ファイルの生成をして終わりでしたが、typed-css-modules には CSS Module の内容が変更されたタイミングで自動生成しなおしてくれる[w オプション](https://github.com/Quramy/typed-css-modules?tab=readme-ov-file#watch)があります。
変更を検知する w オプションはこの後使用していくので、ご認識お願いします。
その他にもオプションはありますが、ここでは触れません。
気になる方はライブラリにある[README](https://github.com/Quramy/typed-css-modules)を参照してください。
動作について確認できたので、package.json の scripts に以下のスクリプトを登録しておきます。

```json
"sass:watch": "tcm src -w",
```

これで変更を検知しつつ、CSS Module ファイルの実装に合わせて型定義ファイルを生成してくれます。

## SCSS の設定と SCSS→CSS→ 型定義ファイルを確立する

ここでは CSS の記載をより楽にするために SCSS について見ていきます。
まずは SCSS の説明をします。
SCSS は CSS を拡張したプリプロセッサであり、変数、ネスト、ミックスイン、継承などの機能を追加することで、より効率的で保守性の高いスタイルシートを作成することができます。
SCSS が有する機能の一部についてサンプルコードを示します。
**変数定義**
変数 SCSS では変数を定義し、再利用することができます。これにより、一貫性のあるスタイルを保ち、メンテナンスを容易にします。

```scss
$primary-color: #007bff;
.button {
  background-color: $primary-color;
}
```

**クラスのネスト**
SCSS ではセレクタをネストすることができ、より読みやすく構造化されたスタイルシートを作成できます。

```scss
.navigation {
  ul {
    margin: 0;
    padding: 0;
    list-style: none;
  }
  li {
    display: inline-block;
  }
  a {
    display: block;
    padding: 6px 12px;
    text-decoration: none;
  }
}
```

**定義の再利用**
ミックスインを使用すると、再利用可能なスタイルのブロックを定義し、必要な場所で呼び出すことができます。

```scss
@mixin border-radius($radius) {
  -webkit-border-radius: $radius;
  -moz-border-radius: $radius;
  -ms-border-radius: $radius;
  border-radius: $radius;
}
.box {
  @include border-radius(10px);
}
```

サンプルコードのように引数も設定できます。
なので[こちらの記事](https://qiita.com/takeuchi_tsuyoshi/items/5c5f5a35845db1551b41)のように、メディアクエリとの併用で画面幅毎のスタイルを簡単に定義できます。
**セレクタの継承**
SCSS では、セレクタ間で共通のスタイルを継承することができます。これにより、コードの重複を減らし、スタイルシートをより簡潔にできます。

```scss
%message-shared {
  border: 1px solid #ccc;
  padding: 10px;
  color: #333;
}
.success {
  @extend %message-shared;
  border-color: green;
}
.error {
  @extend %message-shared;
  border-color: red;
}
```

**演算機能**
SCSS では、数値に対して演算子を使用することができます。これにより、動的な値を生成し、スタイルシートをより柔軟にできます。

```scss
$base-font-size: 16px;
h1 {
  font-size: $base-font-size * 2;
}
h2 {
  font-size: $base-font-size * 1.5;
}
```

以上が SCSS の主な特徴の一部です。
CSS だけだと冗長になってしまったり、ネストの定義が分かりにくくなったりという不都合がありますが、SCSS を使用することで大分解消される可能性があります。
このように SCSS は CSS を構造的に書くのに便利です。
ただ、SCSS そのままでは使用することができず CSS に変換する必要があります。
なので、次に CSS へ変換する方法について見ていきます。

### SCSS を CSS にコマンドで変換する

SCSS を使用するための方法は色々ありますが、今回は後の展開も考えて[Dart Sass](https://sass-lang.com/documentation/cli/dart-sass/)を導入します。
Dart Sass を導入することで CLI で SCSS を CSS に変換できます。
早速やってみます。
まず`npm i -D sass`で Dart Sass をインストールします。
次に以下のいずれかを実行します。
① 単一の SCSS ファイルを CSS に変換する場合
`sass <input.scss> [output.css]`
② 複数 SCSS ファイル or ディレクトリ配下の SCC ファイルを CSS に変換する場合
`sass [<input.scss>:<output.css>] [<input/>:<output/>]`
複数ファイルについては以下のように使用します。

```
# 複数ファイルを一度に変換
$ sass light.scss:light.css dark.scss:dark.css
# ディレクトリ単位での入出力
$ sass themes:public/css
```

基本的には上記のように使用します。
そして、Dart Sass には様々なオプションがあります。
今回は SCSS の変更を検知し、自動で CSS ファイルを再生成する[watch オプション](https://sass-lang.com/documentation/cli/dart-sass/#watch)とソースマップを出力しない[no-source-map オプション](https://sass-lang.com/documentation/cli/dart-sass/#no-source-map)を使用します。
以上のことから、今回使うコマンドを以下のように作成します。

```
$ sass src/:src/ --no-source-map -w
```

これで SCSS から CSS への変換部分の準備が完了しました。

### SCSS→CSS→ 型定義ファイルの流れを構築

最後に SCSS で開発しつつ、型定義ファイルを自動で生成するようにします。
といっても、やることは SCSS でのコンパイルと、CSS Module から型定義ファイルの生成を組み合わせるだけです。
よって、以下のコマンドを package.json のスクリプトに登録するだけです。

```json
"sass:watch": "sass src/:src/ --no-source-map -w & tcm src -w"
```

SCSS を CSS に変換する Dart Sass を watch モードで起動しつつ、CSS Modules から型定義ファイルを作成する typed-css-modules も同様に watch モードで起動しています。
なお、今回は SCSS ファイルは src ディレクトリ配下に存在し、CSS ファイルや型定義ファイルは SCSS と同階層に出力するようにしています。
他のディレクトリにファイルが存在する場合や、出力先を変更した場合は sass コマンドや tcm コマンドの src 部分を適宜入れ替えてください。
注意点としてはそれぞれのコマンドを&&では&で繋いでいる点です。
これによって、二つのコマンドを並列で実行でき、二つの変更検知モードを有効にしつつ起動できます。
以上で準備が完了です。
後は動作確認をしてみましょう。
src ディレクトリ配下に任意の SCSS ファイルを作成します。

```scss
// Test.module.scss
.test {
  .test-part2 {
    font-size: larger;
  }
  .test-part3 {
    font-size: medium;
  }
  .test-part4 {
    font-size: small;
  }
}
```

そして、`npm run sass:watch`を実行します。
すると以下の CSS や型定義ファイルが作成されます。

```css
/* Test.module.css */
.test .test-part2 {
  font-size: larger;
}
.test .test-part3 {
  font-size: medium;
}
.test .test-part4 {
  font-size: small;
}
```

```tsx
// Test.module.d.ts
declare const styles: {
  readonly test: string;
  readonly "test-part2": string;
  readonly "test-part3": string;
  readonly "test-part4": string;
};
export = styles;
```

それぞれ watch モードなので、例えば SCSS ファイルのクラス名を追加したり、変更してみてください。
すると CSS ファイルや型定義ファイルがその変更を反映された値になると思います。
ただ、たまに設定のタイミングが合わず適切に反映されない場合があります。
その時は SCSS を繰り返し保存すれば反映されますので、その点だけはご注意ください。
以上で SCSS を使いつつ補完が効く方法を確立できました。
SCSS で開発をしつつ、型補完によってクラス名の誤りを防ぐことができるので、是非 SCSS を使用し、CSS Modules 形式で開発する場合は今回紹介した方法を検討してください。

## おわりに

今回は SCSS を使いつつ、クラス名に補完が効くようにしました。
SCSS で構造的な実装を CSS で達成しつつ、誤ったクラス名を要素側で設定してしまうことを防ぐことができ結構便利だと感じています。
React や Vue とかで、CSS を結構がっつり書かないと行けなくなった場合に検討していただければ幸いです。
ここまで読んでいただきありがとうございました。
