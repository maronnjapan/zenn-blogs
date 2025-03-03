---
title: "Next.jsのpublicディレクトリにVitepressのビルドファイルをセットする"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NextJS","Vitepress"]
published: true 
published_at: 2025-02-27 
---
## はじめに
マークダウンライクでかつ、いい感じのデザインでドキュメントを作成できるツールに[Vitepress](https://vitepress.dev/)があります。
デフォルトのテーマを使い、マークダウンで書いくだけでいい感じのドキュメントが作れます。
さらには、Vueでも実装ができるので、カスタマイズもしやすいものとなっています。
今回はそのVitepressで記載した内容を、Next.jsを動かしているサーバー上で表示させる方法について見ていきます。
正直Vitepress用にサーバーを立てるのが面倒だっただけで、この方法にそれ以上の利点があるかはわかりません。
Vitepress用に実行環境を用意できない事情がある方の役に立てば幸いです。
## 結論
以下のようにすれば、Next.js上でVitepressにアクセスできるようになります。
1. Next.jsのプロジェクトを立ち上げる(Pages Router,App Routerどちらでも可)
2. `npm add -D vitepress`でVitepressをインストール
3. `npx vitepress init`を実行して、Vitepressのディレクトリを作る(色々聞かれますが、[ドキュメント](https://vitepress.dev/guide/getting-started#setup-wizard)通りにやれば問題ないです)
4. 作成したVitepressのディレクトリ内の.vitepress/config.mtsのdefineConfig関数に以下のオプションを追加

```tsx
export default defineConfig({
  base: '/docs',
  outDir: '../public/docs',
  /** 略 */
})
```
## Vitepressについて
Vitepressはドキュメントを確認すると、以下のものになっています。
> VitePress is a [**Static Site Generator**](https://en.wikipedia.org/wiki/Static_site_generator) (SSG) designed for building fast, content-centric websites. In a nutshell, VitePress takes your source content written in [**Markdown**](https://en.wikipedia.org/wiki/Markdown), applies a theme to it, and generates static HTML pages that can be easily deployed anywhere.

雑に要約すると、Markdownで書いたものをHTMLなどの静的ファイルへ生成し、利用するStatic Site Generation(SSG)となります。
Markdownの記載方法だけでなく、Vueを使った方法も[ドキュメント](https://vitepress.dev/guide/using-vue)に記載されています。
今回は、Vitepressのカスタマイズがメインではないので、説明はこの程度にします。
## Next.jsで静的ファイルを配信する
Next.jsには[ドキュメント](https://nextjs.org/docs/app/getting-started/images-and-fonts)を確認すると、静的ファイルを表示することができるディレクトリが存在します。
デフォルトではpublic ディレクトリであり、実際プロジェクトを作成すると以下のようなファイルたちが存在していると思います。
![2025-02-19_18h26_09.png](/images/move-vitepress-nextjs-public-dir/2025-02-19_18h26_09.png)
他にも任意の静的ファイルを置いてみて、/ファイル名とURLに打ち込めばアクセスできるようになります。
このように、Next.jsはpublicディレクトリに静的ファイルを配置すれば、Next.jsを起動した時にアクセスできるようになります。
以上、VitepressとNext.jsが持つ機能の一部を簡単に見ていきましたので、それらの組み合わせをしてタイトルのことを達成します。
## Vitepressで作成した静的ファイルを表示させる
### 前提
Next.jsのプロジェクトが存在しているものとします。
[ドキュメント](https://nextjs.org/docs/app/getting-started/installation)などを参照にプロジェクトを作っておいてください。
なお、App Router、Pages Routerどちらでも表示できることは確認できています。
### Vitepressの導入
まずは、Vitepressを導入していきます。
といっても、正直[ドキュメント](https://vitepress.dev/guide/getting-started)通りにやるだけです。
なので、詳細は解説しませんが今後の説明では、npmでの操作をしており、`npx vitepress init`を実行した時に表示される質問の一部は以下のように記載しているのを前提にお話します。
- Where should VitePress initialize the config?
    - →./docsで記載しています。
- Use TypeScript for config and theme files?
    - →Yesとしています。
- Add VitePress npm scripts to package.json?
    - →Yesとしています。

導入ができたら、`npm run docs:build`を実行して、docs/.vitepress/distに以下のようなファイルができていればOKです。
![2025-02-22_12h08_16.png](/images/move-vitepress-nextjs-public-dir/2025-02-22_12h08_16.png)
### ビルドの静的ファイルをNext.jsのpublicディレクトリに配置する
ビルドで静的ファイルを作成することができたので、このファイル群をNext.jsが有している静的ファイルへのアクセスができるディレクトリに配置します。
といっても、簡単でdocs/.vitepress/config.mtsのdefineConfig関数に以下のオプションを追加するだけです。
```tsx
import { defineConfig } from 'vitepress'
export default defineConfig({
  outDir: '../public',
  /** 略 */
})
```
こうすることで、Next.jsのpublicディレクトリにビルドした静的ファイルを配置することができます。
では、実際にVitepressのビルドを行い、その後Next.jsを実行してみます。
そして、http://localhost:3000/index.html にアクセスすると以下のような画面が表示されます。
![2025-02-22_12h16_03.png](/images/move-vitepress-nextjs-public-dir/2025-02-22_12h16_03.png)
上手く表示されましたね。
ただ、この場合publicディレクトリを見ると、既存の静的ファイルが消えています。
publicディレクトリには、画像とかもおきたいので、publicディレクトリの中身が上書きされてしまうのは避けたいです。
そうなると、ビルドの出力先はpublicの中にディレクトリを切り、そこに入れたいと思います。
まず思い浮かぶのが、defineConfig関数内の、outDirオプションを`outDir: '../public/docs’`と変更することです。
これは部分的には上手くいきます。
試しにやってみた結果が以下の通りです。
![2025-02-22_12h23_36.png](/images/move-vitepress-nextjs-public-dir/2025-02-22_12h23_36.png)
ファイルにアクセスはできていますが、上手く画面が表示されていません。
この要因を確認するために、ビルドで作成されたファイルの一部を確認してみます。
```html
<!DOCTYPE html>
<html lang="en-US" dir="ltr">
  <head>
    <link rel="preload stylesheet" href="/assets/style.BOSx6zLo.css" as="style">
    <link rel="preload stylesheet" href="/vp-icons.css" as="style">
    
    <script type="module" src="/assets/app.B8SDXkgN.js"></script>
    <link rel="preload" href="/assets/inter-roman-latin.Di8DUHzh.woff2" as="font" type="font/woff2" crossorigin="">
    <link rel="modulepreload" href="/assets/chunks/theme.CFzztjJw.js">
    <link rel="modulepreload" href="/assets/chunks/framework.DN7zo1cr.js">
    <link rel="modulepreload" href="/assets/index.md.CMA4bbig.lean.js">
  </head>
</html>
```
注目すべき点はパスの設定部分です。
基本的に、パスが/assetsから始まっています。
publicファイルにある上記ファイルにアクセスした場合、対象のパスはhttp://localhost:3000/assets/~ という形でで取得しようとします。
しかし、先程outDirの設定でファイルの出力はすべてpublic/docsに配置されています。
なので、http://localhost:3000/assets/~ ではなく、http://localhost:3000/docs/assets/~ という形でファイルへのアクセスが必要です。
このずれが、画面の表示をおかしくしていました。
つまり、HTMLファイルにはアクセスできても、その中で読み込もうとしている他のファイルにアクセスができなかったため、表示が上手くいっていませんでした。
よって、いい感じに表示させるには/assetsの前に/docsを足す必要がありますが、それを手動でやるのはしんどいです。
そこで、Vitepressの設定の一つである[baseオプション](https://vitepress.dev/reference/site-config#base)を使用します。
baseオプションはざっくりいうと、Vitepressのルートファイルにアクセスする時のパスを設定するものです。
デフォルトは`/`なのですが、例えばドキュメントにあるようにに`/bar/`を設定すると、http://example.com/bar/ にアクセスした時、Vitepressのルートファイルにアクセスするようになります。
そして、このbaseオプションに値を設定すると、ビルドしたファイルも    `<link rel="modulepreload" href="/bar/assets/index.md.BTKG5DAH.lean.js">`とパスの先頭に設定した値を付与します。
この機能を今回も使用します。
docs/.vitepress/config.mtsの設定を以下のように変更します。
```tsx
import { defineConfig } from 'vitepress'
export default defineConfig({
  base: '/docs/',
  outDir: '../public/docs',
  /** 略 */
})
```
この状態で、Vitepressをビルドします。
その後、Next.jsを起動しhttp://localhost:3000/docs/index.html にアクセスします。
すると、以下のように画面がいい感じに表示されるようになりました。
![2025-02-22_12h45_14.png](/images/move-vitepress-nextjs-public-dir/2025-02-22_12h45_14.png)
以上で、Next.jsの静的ファイルを配信するディレクトリを活用して、Vitepressで作成した画面を表示することができました。
## おわりに
今回は、Next.jsのpublicディレクトリ内にVitepressの静的ファイルを配置して、Vitepress用にサーバーを立てずとも表示できる方法をみていきました。
やり方さえ知ってしまえば、作業は単純なのでとりあえずサーバー上で表示させたい時などに試していただければと思います。
ただ、ここまで書いておいてなんですが、正直こんなことをせずにVitepress用にサーバーを建てた方がいいと思います。
昨今では、例えば[Cloudflare](https://www.cloudflare.com/ja-jp/)など簡単に静的サイトを作成できるサービスは溢れています。
そのため、どうしてもNext.jsを動かしているサーバーしか用意できない場合など、特殊なケース以外は順当にサーバーをたてることを推奨します。
まあ、こんな小ネタもあるということを認識いただけると幸いです。
ここまで読んでいただきありがとうございました。