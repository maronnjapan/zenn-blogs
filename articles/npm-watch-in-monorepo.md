---
title: "モノレポチックなプロジェクトで、ファイル変更を監視しながらサーバーの並列実行を行う"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["monorepo", "npm"]
published: true
---

# モノレポチックなプロジェクトでアプリケーションを並列実行する

## はじめに

複数のリポジトリを一つのプロジェクトで管理する方法として、monorepo というものがあります。
複数のリポジトリをまとめることで、それぞれリポジトリを別で作りやすく管理がしやすくなります。
そして、一つにまとめたことでルートプロジェクトから各プロジェクトを操作できれば、それぞれアプリケーションを起動して、コードの変更があったらそれを反映させるために再度各アプリケーションを起動する必要がなくなりそうです。
今回はそんなお話です。

## プロジェクト構築

npm workspace を使ったモノレポプロジェクトを作成する方法は[npm workspace を利用して NestJS + Create React App をモノレポ化しよう](https://note.com/shift_tech/n/nbecb007ac2ee)という良記事があるので、そちらを参照してください。
私は上記記事を参考にして、以下のようなディレクトリ構造のプロジェクトを作成しました。
.
├── backend(NestJS) 
├── frontend(React)  
└── packages
└── ts-router(ts-rest)
なお、この先コードのビルドコマンドとして`npm run compile`が設定されていることを前提にして話を進めます。
ご認識お願いします。

## ルートプロジェクトから各アプリケーションを起動する

ここからは、先程 npm workspace を使用して作成したプロジェクトをそれぞれルートプロジェクトから起動できるようにします。
といっても、とても簡単で、npm コマンドの最後に w オプションを付与するだけで、各プロジェクト内のコマンドを実行できます。
もう少し具体的に見ていきます。
まず、NestJS と React でそれぞれサーバーを起動するコマンドを確認します。
NestJS の場合は`npm run start:dev`で React の場合は`npm run dev`でした。
そのため、ルートプロジェクトでもこれらのコマンドを実行できるようにすれば目的を果たせます。
そして、それを達成するにはルートプロジェクトの package.json に以下のスクリプトを登録すれば良いです。

```json
"scripts": {
	"start:dev:api": "npm run start:dev -w @monorepo-oidc-app/backend",
	"start:app": "npm run dev -w @monorepo-oidc-app/frontend"
},
```

それぞれ、NestJS や React を起動するコマンドを設定していますが、後ろに w オプションがあり、値として各プロジェクトの package.json で設定した名前を付与しています。
これでルートプロジェクトから各プロジェクトを起動するコマンドが作成できました。
後は、ルートプロジェクト配下で`npm run start:dev:api`や`npm run start:app`と入力すれば NestJS と React が起動できます。

## NestJS と React を同時に起動する

先程設定したスクリプトで NestJS、React をそれぞれ動かせるようになりました。
ただ、NestJS と React 両方を動かすにはそれぞれターミナルを立ち上げて、作成したスクリプトを実行する必要があります。
これは正直手間です。
いちいち複数のターミナルを立ち上げる手間もありますし、ターミナルが複数になることでどのターミナルで何をしているかが見にくくなります。
なので、ここでは一個のコマンドで NestJS と React を同時に起動させるようにします。
と、前置きを長々と書きましたが、上記目的達成するには以下の script をルートプロジェクトの package.json に登録するだけで完了します。

```json
"start":"npm run start:app & npm run start:dev:api"
```

&が一個足らなく見えますが、これでも npm コマンドは動きます。
皆さんがよく見る&&は左側のコマンド処理が実行完了したら、右側のコマンドを実行するといったコマンドを直列実行する際に使用します。
一方、&一つだけの場合は左右それぞれのコマンドを並列で実行させることを意味します。
すなわち、&で繋いだ場合 2 つのコマンドを同時に実行することができます。
これはまさに今回の目的を達成するものとなります。
実際に上記スクリプトをルートプロジェクトで実行すると、まず React が立ち上がり、React のサーバーが実行されたまま NestJS が起動します。

## ts-rest の変更を検知して、再ビルドと再起動させる

これまでで、ルートプロジェクトから NestJS と React を実行するための w オプションや、並列で実行するための&コマンドについて見てきました。
これらによって、一つのターミナルで複数のアプリケーションを動かせるようになっています。
しかし、packages フォルダに作成した ts-rest のコードはホットリロードなどの機能はありません。
そのため、仮に ts-rest まわりの定義を変更したとしても、再度コードをビルドしてからサーバーを起動させないと NestJS,React に反映されません。
これは開発体験がよくありません。
そこで、ここでは packages 内の変更を検知して、再ビルド＆サーバーの再起動を行うようにします。
まずファイル変更検知を行ってくれる[npm-watch](https://github.com/M-Zuber/npm-watch)モジュールを、ルートプロジェクト配下で`npm install npm-watch`を実行してインストールします。
インストール後、ルートプロジェクトの package.json に以下のコードを記載します。

```json
"watch": {
        "restart": {
            "patterns": [
                "packages/ts-router"
            ],
            "extensions": "ts",
            "ignore": "packages/ts-router/dist/*"
        }
    },
```

そして、同様の package.json に以下の scripts を登録します。

```json
"restart": "npm run compile &&  ( npm run start:app & npm run start:dev:api)"
"watch": "npm-watch"
```

それぞれについて、説明します。
まず、watch オブジェクト周りについてです。
watch プロパティは npm-watch がどういったファイル変更を検知するかを定義するためのプロパティ名となっています。
このプロパティ名を設定しないと、エラーが発生しコマンドが終了します。
次に restart についてです。
これは変更が検知されたときに実行する scripts 名を設定します。
なので、scripts に設定した`restart`を記載しています。
restart の中身は以下の値を定義しています。

- patterns

変更を検知する対象のディレクトリやファイルを配列指定できます。
ワイルドカードなども使用できます。

- extensions

変更を検知するファイルのパスを指定します。

- ignore

変更を検知する対象としないディレクトリやファイルを指定できます。
今回でいうと、ビルドした出力先である dist 配下のコードも対象にしてしまうと、一生変更を検知し続けてしまうので、検知対象から外しています。
これで npm-watch を動かす準備ができたので、後は`npm run watch` を実行すれば最初`npm run restart`の中身が実行され、以降 ts-rest のコードを変更するたびに`npm run restart`が再実行されます。
そして、restart スクリプトの中身はプロジェクト全体をビルドした後、NestJS と React を並列で起動するものとなっています。
以上のことから、ts-rest が変更されると再ビルドとサーバーの再起動が行われるので、ts-rest の定義が常に最新の状態でアプリケーションを起動することができます。
これで一個のコマンドで、コードを変更するたび常に最新の状態でアプリケーションを操作できるようになりました。
便利な時代になったものです。是非お試しください。

## 余談 npm-watch モジュールはなにをしているの？

npm-watch モジュールは[nodemon モジュール](https://github.com/remy/nodemon)をラップしたモジュールとなっています。
それを裏付けるコードが以下の部分です。

```tsx
var npm = process.platform === "win32" ? "npm.cmd" : "npm";
var nodemon = process.platform === "win32" ? "nodemon.cmd" : "nodemon";
//...略
var proc = (processes[script] = spawn(nodemon, args, {
  env: process.env,
  cwd: pkgDir,
  stdio: inherit === true ? ["pipe", "inherit", "inherit"] : "pipe",
}));
```

一部略した部分で定義した変数もあるので、このコードだけでは動きませんが、npm-watch モジュールが何をしたいかは上記コードだけで伝わります。
「オプションを付与しながら nodemon コマンドを実行する」、これが npm-watch モジュールが行っていることです。
その他のコードは package.json に設定した値を基に、オプションを生成する処理とターミナルにログを出力するための処理を行っています。
これらのことから、npm-watch の機能が分からない時は npm-watch だけでなく、nodemon のドキュメントと一緒に確認するとより機能の理解が深まります。

## 余談 npm-watch を typescript で作り直す

以下のような bin ファイルと nodemon を実行するファイルを作りました。

```tsx
//watch.ts
//nodemonを実行するための関数などを定義
import { spawn } from "child_process";
import { obj } from "through2";
import * as internal from "stream";
type WatchIndividualConfig = {
  scriptName: string;
  options?: {
    verbose?: boolean;
    delayTime?: number;
    legacyWatch?: boolean;
    ignorePaths?: string[];
    watchPaths?: string[];
    watchExtensions?: (
      | "py"
      | "php"
      | "java"
      | "css"
      | "html"
      | "ts"
      | "hsl"
      | "rb"
    )[];
  };
};
type WatchGlobalConfig = {
  setMaxListeners: boolean;
};
export type WatchConfig = {
  watchConfig: WatchIndividualConfig[];
  watchGlobalConfig?: WatchGlobalConfig;
};
export type ReturnFunction = (func?: (code?: number) => never) => {
  stdin: internal.Transform;
  stderr: internal.Transform;
  stdout: internal.Transform;
};
export const difineWatchConfig = (config: WatchConfig) => config;
const stdObj = {
  stderr: obj(),
  stdout: obj(),
  stdin: obj(function (line, _, callback) {
    line = line.toString();
    var match = line.match(/^rs\s*(.*)/);
    if (!match) {
      console.log("Unrecognized input:", line);
      return callback();
    }
    callback();
  }),
};
export function watchPackage(
  pkgDir: string,
  exit: (code?: number) => never,
  pkgs: WatchIndividualConfig[],
  globalConfig?: WatchGlobalConfig
) {
  var scriptsCount = pkgs.length;
  pkgs.forEach(function (script) {
    if (!script.scriptName) {
      die('No such script "' + script + '"', 2);
    }
    startScript(
      script.scriptName,
      pkgDir,
      script,
      globalConfig?.setMaxListeners ?? false,
      scriptsCount + 1
    );
  });
  return stdObj;
  function die(message: string, code: number) {
    process.stderr.write(message);
    stdObj.stdin.end();
    stdObj.stderr.end();
    stdObj.stdout.end();
    exit(code || 1);
  }
}
function startScript(
  script: string,
  pkgDir: string,
  pkg: WatchIndividualConfig,
  setMaxListeners: boolean,
  scriptsCount: number
) {
  const npm = process.platform === "win32" ? "npm.cmd" : "npm";
  const nodemon = process.platform === "win32" ? "nodemon.cmd" : "nodemon";
  const exec = [npm, "run", "-s", script].join(" ");
  const watchPaths = pkg.options?.watchPaths
    ? pkg.options.watchPaths.map((path) => ["--watch", path]).flat()
    : [];
  const extensions = pkg.options?.watchExtensions
    ? ["--ext", pkg.options.watchExtensions.join(",")]
    : [];
  const ignorePaths = pkg.options?.ignorePaths
    ? pkg.options.ignorePaths.map((path) => ["--ignore", path]).flat()
    : [];
  const legacyWatch = pkg.options?.legacyWatch ? ["--legacy-watch"] : [];
  const delay =
    pkg.options?.delayTime && pkg.options?.delayTime > 0
      ? ["--delay", pkg.options.delayTime + "ms"]
      : [];
  const verbose = pkg.options?.verbose ? ["-V"] : [];
  if (verbose && silent) {
    console.error("Silent and Verbose can not both be on");
  }
  const args = [
    ...["--exec", exec],
    ...watchPaths,
    ...extensions,
    ...ignorePaths,
    ...legacyWatch,
    ...delay,
    ...verbose,
    ...silent,
  ];
  if (setMaxListeners) {
    process.setMaxListeners(scriptsCount);
    stdObj.stdout.setMaxListeners(scriptsCount);
    stdObj.stderr.setMaxListeners(scriptsCount);
  }
  const proc = spawn(nodemon, args, {
    env: process.env,
    cwd: pkgDir,
    // これを有効にしないとnodemonを起動しても、ログが表示されない
    stdio: ["pipe", "inherit", "inherit"],
  });
}
```

```tsx
#!/usr/bin/env node
//cli.ts
//ターミナルで「npm-watch-by-ts」を実行可能にするbinファイル
import * as path from "path";
import { WatchConfig, watchPackage } from "./watch";
import { writeFileSync } from "fs";
// 実行している環境がWindows環境かをチェックしている
const windows = process.platform === "win32";
// 環境変数のPATHもしくは、Pathの値をprocess.envから取得するためにOSごとに合わせた文字列を取得している
const pathVarName = windows && !("PATH" in process.env) ? "Path" : "PATH";
// 環境変数のPATH or Pathにnode_modulesの.binフォルダまでのパスを追加している。
// PATH周りについてはhttps://qiita.com/sta/items/63e1048025d1830d12fd を参考にすること
process.env[pathVarName] +=
  path.delimiter + path.join(__dirname, "node_modules", ".bin");
// `npm-watch-by-ts init`というコマンドを実行したら、対象プロジェクトに設定ファイルを生成させる。
if (process.argv[2] === "init") {
  writeFileSync(
    path.join(process.cwd(), "watch-init.ts"),
    `
import { difineWatchConfig } from 'npm-watch-by-ts/dist/watch'
// By default, npm-watch-by-ts looks for files with the .js, .mjs, .coffee, .litcoffee, and .json extensions.
// If you want npm-watch-by-ts to looks for other files, add watchExtensions options.
export default difineWatchConfig({
    watchConfig: [{
        scriptName: 'Script Name you want to exec in package.json',
        options: {}
    }]
})
`
  );
  process.stdout.write("Create watch-init.ts");
  process.exit(-1);
}
// プロジェクト側で設定ファイルをコンパイルしたものをインポートして、スクリプトを実行している。
import(path.join(process.cwd(), "watch-init.js"))
  .then((module) => {
    const config: WatchConfig = module.default;
    if (!config) {
      process.stderr.write(`Not Found Watch Module\r\n`);
      process.exit(-1);
    }
    const watcher = watchPackage(
      process.argv[3] || process.cwd(),
      process.exit,
      config.watchConfig,
      config.watchGlobalConfig
    );
    process.stdin.pipe(watcher.stdin);
    watcher.stdout.pipe(process.stdout);
    watcher.stderr.pipe(process.stderr);
  })
  .catch(() => {
    process.stderr.write(
      `Not Found watch-init.js\r\n please exec 'npx tsc watch-init.ts'`
    );
    process.exit(-1);
  });
```

後はこれをモジュール化して、`npx モジュール名 init`で watch-init.ts を作った後、`npx watch-init.ts`で設定ファイルをコンパイルし、npx モジュール名で`npm-watch`を使った場合と同じ動作ができるようになります。
設定ファイルをコンパイルしないと使えないという非常に大きな欠点はありますが、nodemon を実行する際のオプションに型補完を効かせることができたので、一旦ここで満足しました。
Vite は vite.config.ts の内容を特にコンパイルせずにインポートしているので、一体どうやっているのか凄く気になります。
機能自体は npm-watch で説明した通り、以下の spawn メソッドを使用して、nodemon コマンドを実行するのが主となっています。

```tsx
const proc = spawn(nodemon, args, {
  env: process.env,
  cwd: pkgDir,
  // これを有効にしないとnodemonを起動しても、ログが表示されない
  stdio: ["pipe", "inherit", "inherit"],
});
```

そのため、機能自体はあまり説明することがありません。
一方で、以下の型で定義した各オプションが nodemon のどの機能と一致するかはパッと見分からないので、こちらは`legacyWatch`以外解説します。
legacyWatch オプションはいまいち使いどころがわからなかったので、割愛します。
気になる方は[ドキュメント](https://github.com/remy/nodemon/tree/main?tab=readme-ov-file#application-isnt-restarting)を参照ください。すみません。

```tsx
type WatchIndividualConfig = {
  scriptName: string;
  options?: {
    verbose?: boolean;
    delayTime?: number;
    legacyWatch?: boolean;
    ignorePaths?: string[];
    watchPaths?: string[];
    watchExtensions?: (
      | "py"
      | "php"
      | "java"
      | "css"
      | "html"
      | "ts"
      | "hsl"
      | "rb"
    )[];
  };
};
```

### verbose オプション

nodemon で js 系以外のスクリプトを実行したいときに使用します。

### delayTime オプション

これは変更を検知してから、指定のスクリプトを起動するまでの時間を遅らせたいときに設定するオプションです。

### ignorePaths オプション

変更検知の対象外にしたいパスを設定します。
/src/index.js とファイル名も指定できますし、src/とディレクトリ名も指定できます。
ディレクトリ名を指定したときは配下のファイルが全て対象となります。
また、ファイル名、ディレクトリ名にはワイルドカードも使用できます。

### watchPaths オプション

先ほどは変更検知の対象外を選びましたが、こちらは変更検知の対象とするパスを設定します。
ignorePaths オプションと同様にワイルドカードを使用できます。

### watchExtensions オプション

nodemon は`.js`, `.mjs`, `.coffee`, `.litcoffee`, `.json`のパスしかデフォルトで検知しません。
そのため、上記ファイル以外の変更を検知して欲しい場合はこのオプションに変更を検知したファイルの拡張子を記載します。
なお、私が記載している拡張子が限定的なのは特に意味はないです。
単に様々な拡張子を記載するのが面倒だったためです。

## おわりに

今回はモノレポプロジェクトで各アプリケーションを同時に実行しつつ、変更を検知できるようにしました。
これによって、開発する際`npm run watch`さえ実行すれば、コードを変更するたびコマンドを再度実行する必要がなくなりました。
これで、ストレスなく開発ができそうです。
