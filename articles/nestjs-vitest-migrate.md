---
title: "VitestでNestJSのテストを実行する方法とプラグインについて軽く触れる"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NestJS","Vitest"]
published: true
---
## はじめに
NestJSはテストツールとして標準でjestを設定しています。
ただ、今回は実行速度を求めて[Vitest](https://vitest.dev/)に置き換えていきます。
その手順や、個人的なポイントを書いていきます。
ただし、Vitestは何たるかやVitestを導入することの利点などは記載しません。
この記事はVitestを導入する時のポイントという観点で書いておりますので、そもそも導入するべきか否かについては別記事を参照いただけますと幸いです。
## 導入手順
以下の手順で導入できます。
### 最小構成
- `npm i --save-dev vitest unplugin-swc`でパッケージをインストール
- 以下vitest.config.(m)tsで記載する。

```tsx
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
    plugins: [swc.vite()],
});
```
- package.jsonにあるjestコマンドを`vitest`に変更する
    - watchモードで動かしたくない場合、`vitest run`にする。
    - 必要な範囲を一部抽出したのが以下部分です。

```tsx
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
  },
  "devDependencies": {
    "unplugin-swc": "^1.5.2",
    "vitest": "^3.1.3"
  },
}

```
### tsconfig.jsonでエイリアスを設定している場合
- `npm i --save-dev vitest unplugin-swc vite-tsconfig-paths`でパッケージをインストール
- 以下vitest.config.mtsで記載する。
    - 必ず拡張子は**mts**にすること。

```tsx
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [
    tsconfigPaths(),
    swc.vite({
      module: { type: 'es6' },
    }),
  ],
});

```
- package.jsonにあるjestコマンドを`vitest`に変更する
    - ここは最小構成の時と同じです。

手順はこれだけで動きます。
後は個人的なポイントを書いていくので、気になる部分を適宜読んでいただければと思います。
## プラグインの必要性
:::message
※ここの解説は推測が混じっている部分もあります。Vitestを動かすことはできていますが、それ以外に関しては内容の正確性を担保するものではないことはご了承ください。
:::
ここではなぜ`swc.vite`プラグインが必要か見ていきます。
結論としては、esbuildによるトランスパイルなどを防ぐためです。
順を追ってみていきます。
実は、NestJSでJestからVitestに置き換えた時、テスト自体はすぐに実行できます。
そして、コードによってはテストも成功します。
例えば、以下のコードがあります。
```tsx
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  private readonly appService: AppService;
  constructor() {
    this.appService = new AppService();
  }

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}

```
AppServiceのコードは以下の通りです。
```tsx
import { Injectable } from '@nestjs/common';

@Injectable()
export class AppService {
  getHello(): string {
    return 'Hello World!';
  }
}
```
これに対して以下のテストコードがあります。
```tsx
import { Test, TestingModule } from '@nestjs/testing';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { describe, beforeEach, it, expect } from 'vitest';

describe('AppController', () => {
  let appController: AppController;

  beforeEach(async () => {
    const app: TestingModule = await Test.createTestingModule({
      controllers: [AppController],
      providers: [AppService],
    }).compile();

    appController = app.get<AppController>(AppController);
  });

  describe('root', () => {
    it('should return "Hello World!"', () => {
      expect(appController.getHello()).toBe('Hello World!');
    });
  });
});
```
これをVitestで実行するとテストは成功します。
ならプラグインは不要に見えますが、Controllerクラスのコンストラクタ部分を以下のように置き換えてみます。
```tsx
export class AppController {
  constructor(private readonly appService:AppService) {}

  /** 省略 */
}
```
これでVitestによるテストを実行すると以下のようにエラーとなります。
![2025-05-18_16h11_44.png](/images/nestjs-vitest-migrate/2025-05-18_16h11_44.png)
テストは実行できているのに、AppServiceクラスのgetHelloメソッドが存在せずエラーとなっています。
この要因としては大きく以下の二つです。
- Vitestはesbuildによって実行される
- NestJSのデコレーター処理は多くがMetadataを活用する形になっている

まず、Vitestの動作について確認します。
Vittestは内部的にViteを使用して実行される認識です。(※要出典)
そして、Viteですが[Typescriptのトランスパイルをesbuildによって行います](https://ja.vite.dev/guide/features.html#typescript-%E3%82%B3%E3%83%B3%E3%83%8F%E3%82%9A%E3%82%A4%E3%83%A9%E3%83%BC%E3%82%AA%E3%83%95%E3%82%9A%E3%82%B7%E3%83%A7%E3%83%B3:~:text=Vite%20%E3%81%AF%20esbuild%20%E3%82%92%E7%94%A8%E3%81%84%E3%81%A6%20TypeScript%20%E3%82%92%20JavaScript%20%E3%81%AB%E5%A4%89%E6%8F%9B%E3%81%97%E3%81%BE%E3%81%99%E3%80%82)。
このesbuildを用いることでViteはあの速さを体現していますが、esbuildはTypescriptの [`emitDecoratorMetadata`](https://www.typescriptlang.org/tsconfig#emitDecoratorMetadata)を[サポートしていません](https://esbuild.github.io/content-types/#no-type-system)。
emitDecoratorMetadataのあれこれの話は[公式ドキュメント](https://www.typescriptlang.org/docs/handbook/decorators.html#metadata)（[日本語訳](https://mae.chab.in/archives/59845#:~:text=%E5%8F%82%E7%85%A7%E3%81%8F%E3%81%A0%E3%81%95%E3%81%84%E3%80%82-,Metadata%3A%20%E3%83%A1%E3%82%BF%E3%83%87%E3%83%BC%E3%82%BF,-Some%20examples%20use)）に委ねるとして、ここではデコレータを付与したクラス・メソッド・プロパティにはそれぞれ、以下の値をキーにしたものがそれぞれReflectオブジェクトに格納されるという理解でいてください。
- `design:type`(メソッドデコレーターとプロパティデコレータのみ)
- `design:paramtypes`(クラスデコレータとメソッドデコレーターのみ)
- `design:returntype`(メソッドデコレーターのみ)

emitDecoratorMetadataがサポートしていないということはすなわち、上記の値がReflectオブジェクトにセットされないことになります。
次にNestJSですが、`@Injectable`に始まり多くのデコレータが駆使されています。
そして、以下のNestJSのソースコードには`design:paramtypes`が変数に定義されています。
https://github.com/nestjs/nest/blob/eac355f39de25647813ee5cdb6b168412bdd93fb/packages/common/constants.ts#L11
この変数は例えば、クラスを実際に注入する処理が存在するinjector.tsで使用されています。
https://github.com/nestjs/nest/blob/eac355f39de25647813ee5cdb6b168412bdd93fb/packages/core/injector/injector.ts#L389
このようにNestJSではemitDecoratorMetadataがtrueの前提で処理が行われる箇所があります。
そのため、esbuildで動くと当該部分がコンパイルした際に出力されず、上手く動かないということに遭遇します。
Vitestはesbuildで動く、NestJSはemitDecoratorMetadataが有効である前提で実装されている。
この差によるテストが実行できないという問題を埋めるために、`swc.vite()`が存在します。
unplugin-swcはViteやRollupでSWCを使ってコンパイルできるようにしたものになります。
SWCでのコンパイルなどを行うために、unplugin-swcプラグインはViteの[esbuildオプション](https://ja.vite.dev/config/shared-options.html#esbuild)を[無効にするコード](https://github.com/unplugin/unplugin-swc/blob/main/src/index.ts#L107C6-L113C9)を含んでいます。
そして、さらにSWCのコンパイルオプションで[emitDecoratorMetadataオプションなどを有効にする処理](https://github.com/unplugin/unplugin-swc/blob/main/src/index.ts#L73C9-L81C10)もあります。
unplugin-swcは[README](https://github.com/unplugin/unplugin-swc/tree/main?tab=readme-ov-file#tsconfigjson)にもあるように、tsconfig.jsonの設定を引き継ぐので、NestJSを使用していればこれらのMetadata周りの設定は有効になります。
さらに[NestJSのドキュメント](https://docs.nestjs.com/recipes/swc)にもSWCへの変更の記載があることからも、SWCであれば大抵は大丈夫なのかなと思います。
以上のように、unplugin-swcによって、コンパイルなどの設定をSWCに寄せてしまうことで、NestJSとesbuildの相性の悪さを打ち消し、より高速にテストを回すことができます。
ちなみに、SWCのドキュメントには[ベンチマークのページ](https://swc.rs/docs/benchmarks)もあって、esbuildに遜色ないことをうたってます。
なので、そこまでesbuildを使えないことを悲観しなくても良さそうです。
## tsconfigの設定を引き継ぐ
もう一つのプラグインであるtsconfigPathsも軽く見ていきます。
このプラグインは既存のtsconfigの設定をVitestに適用できます。
これだけだと先程のunplugin-swcがtsconfig.jsonを参照しているので、不要そうに見えます。
しかしunplugin-swcはtsconfigの設定すべてを使うわけではないです。
例えばエイリアスは適用されません。
なのでVitestの設定ファイルに書いておく必要があります。
そういった煩わしさを無くすのがtsconfigPathsプラグインとなります。
Vitestを動かすのに必須ではないですが、入れてくと便利かと思います。
なお、このプラグインはESModule専用ライブラリのため、使用の際は`swc.vite({module: { type: 'es6' },})`に設定を上書きする必要があります。
## (余談)Metadataを使用した簡単なDIの実装
最後に、本筋とはほぼ関係ないですが、Typescriptでデコレータを使用してDIを行う方法についてみていきます。
なお、これがNestJSのDIの仕組みと同じであるかは分かりませんので、その点はご了承ください。（きっと同じなんだろうと思いつつも、結局確信がもてませんでした。が、結構調査に時間を使い、悔しいと思ったのでこの節を書いています。）
実装は以下の通りです。
```tsx
import 'reflect-metadata';

/** DIコンテナクラス */
class Container {
    /** コンストラクタ依存関係を解決してインスタンスを取得 */
    resolve<T>(target: any): T {

        /** クラスのメタデータから依存関係情報を取得 */
        const tokens = Reflect.getMetadata('design:paramtypes', target) || [];

        /** 依存関係を再帰的に解決 */
        const injections = tokens.map((token: any) => this.resolve(token));

        /** 引数のクラスをインスタンス化 */
        const instance = new target(...injections);

        return instance;
    }
}

/** Injectableデコレータ関数 */
function Injectable() {
    return function (target: any) { };
}

/** Controllerデコレータ関数 */
function Controller() {
    return function (target: any) { };
}
const container = new Container();

/** サービスクラス*/
@Injectable()
class LoggerService {
    log(message: string): void {
        console.log(`[INFO] ${message}`);
    }
}

@Injectable()
class DatabaseService {
    connect(): void {
        console.log('Connected to database');
    }
}

@Injectable()
class UserService {
    constructor(
        private logger: LoggerService,
        private database: DatabaseService
    ) { }

    getUsers(): string[] {
        this.logger.log('Fetching users');
        this.database.connect();
        return ['User1', 'User2', 'User3'];
    }
}

@Controller()
class UserController {
    constructor(private userService: UserService) { }

    listUsers(): void {
        const users = this.userService.getUsers();
        console.log('Users:', users);
    }
}

function bootstrap() {
    /** DIコンテナからUserControllerを解決（依存関係も自動的に解決される）*/
    const userController = container.resolve<UserController>(UserController);

    /**コントローラーのメソッドを呼び出し */
    userController.listUsers();
}

bootstrap();
```
tsconfig.jsonの設定は必ず以下の内容は含むようにする必要があります。
```tsx
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```
実装で注目するのは、Containerクラスです。
これはReflectオブジェクトに登録された`design:paramtypes`をかき集めて、再帰的にクラスをインスタンス化してコンストラクタに設定しています。
これによって、実装側で明示的にインスタンス化→注入をせずともDIの仕組みを使うことができます。
なぜ、`design:paramtypes`で値が取得できるのかも確認していきます。
まず、先程のTSファイルをコンパイルした結果の一部を以下に示します。
```tsx
var __metadata = (this && this.__metadata) || function (k, v) {
    if (typeof Reflect === "object" && typeof Reflect.metadata === "function") return Reflect.metadata(k, v);
    
var UserController = /** @class */ (function () {
    /** ...略 */
    UserController = __decorate([
        Controller(),
        __metadata("design:paramtypes", [UserService])
    ], UserController);
    return UserController;
}());
```
ファイルを呼び出したタイミングで、UserControllerクラスの設定を行っています。
その中に__metadata関数があり、この関数は`"design:paramtypes"`をキーにUserControllerクラスにDIしているものをReflectオブジェクトに登録しています。
これによって、`Reflect.getMetadata('design:paramtypes', 対象のクラス)`とした際に依存クラスを取得でき、インスタンス化を行うことが可能になります。
自分の中のイメージとしては、共通のキーを使ってそれを取り出すということまで仕組みとしてあるから、後はそれを実装側で良しなにするというのがTypescriptにおけるデコレータを使ったDIの仕組みなるのかなと思いました。
なお、この実装の注意点として必ずそれぞれのクラスにデコレーターの付与が必要です。
デコレータ内の処理はサンプル実装のように空で問題ないのですが、デコレータが付与されていないとReflectオブジェクトの登録対象外となってしまいます。
実際にUserControllerクラスにデコレーターを付けなかった時のコンパイル結果は以下の通りです。
```tsx
var UserController = /** @class */ (function () {
    function UserController(userService) {
        this.userService = userService;
    }
    UserController.prototype.listUsers = function () {
        var users = this.userService.getUsers();
        console.log('Users:', users);
    };
    return UserController;
}());
```
Metadata周りの処理が丸ごと抜けています。
丸ごと抜けているせいで、Reflectオブジェクトへの登録もされないため、上手く依存クラスを取得できません。
なので、Containerクラスによる依存の解決もできず実行するとエラーとなります。
以上簡単なDIの実装でした。
私のこれまでの理解よりも圧倒的にデコレータやMetadataは深いなと感じました。
## おわりに
今回はNestJSをVitestで実行する方法や、各種設定について見ていきました。
自身のVite・Vitest・esbuild・SWC・Metadataなどなどの理解の浅さによって、内容の割に記事を書くのに時間がかかってしまいました。
まだまだ理解は不十分ではありますが、とりあえずNestJSでVitestを動かせてはいるので一旦は良しとし、徐々に理解していこうと思います。
ここまで読んでいただきありがとうございました。