---
title: "デコレーターを使って、クラス内のメソッドを実行する時に値が書き変わる前のプロパティを保存する方法(Typescript使用)"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Javascript", "Typescript"]
published: true
---

# はじめに

タイトルを見るとあまり使用するケースが思い浮かばないかもしれません。
なので、まずはこの記事を書く動機になったケースをお話します。
実装しているアプリケーションでは、通知機能がありました。
この通知は管理者の追加・削除された時、対象の人にいきます。
イメージは以下のような形です。
![event.drawio.png](/images/nestjs-custom-decolator/event.drawio.png)
当初はまず既存の管理者情報を取得し、その人に通知を送り、新しい管理者がセットされたら再度対象の人に通知を送るようにしていました。
しかし、通知処理が走る部分の処理を見直し、以下のようなレスポンスの時に実行されるインターセプター部分で、通知がいくように修正しました。
![event-interceptor.drawio.png](/images/nestjs-custom-decolator/event-interceptor.drawio.png)
このようにインターセプターで処理を行った理由としては、主に二つあります。
一つ目は管理者権限周りのエンドポイントは今後増加する可能性があり、エンドポイントを増やす度に都度同じ処理を実装することを避けたかったためです。
インターセプターに通知処理をまとめてしまうことで、管理者権限周りのエンドポイントが増えても自動で通知がいくようになります。
この辺の実装についても、またいつか記事にはしたいですが、今回の主題とは異なるので詳細は省略させていただきます。
二つ目は通知処理をイベントによって実行させ、主となる追加・削除リクエストとは切り離したかったためです。
切り離しを行うことで、全体のリクエスト完了時間の短縮や、仮に通知周りでエラーが発生しても処理自体は止めないようにしたかったからです。
以上の経緯で通知の実行場所を変更したのですが、問題が発生しました。
それを見ていくために、まず通知対象の人を取得する元々の流れをみていきます。
通知を送る対象の人は以下のクラスの administrators プロパティから抽出していました。

```tsx
class Group {
  administrators: number[];
}
```

元々の処理の流れは以下の通りです。
リクエストに含まれる ID から、現状の状態を値にもった Group クラスを作成する。
→administrators プロパティを取得し、一致するユーザーに通知する
→ リクエストの内容をもとに、Group クラスの administrators プロパティを書き換える
→ 再度 administrators プロパティを取得し、一致するユーザーに通知する。
→Group クラスを永続化する
状態の書き換え前後で通知処理を実行していました。
しかし、通知処理をレスポンス時に実行するインターセプターにまとめてしまったことで、変更後のユーザーしか取得できません。
既存のユーザーに通知したくても、情報がなくてできなくなりました。
そこで、Group クラスを修正し以下のようなプロパティを付与しました。

```tsx
class Group {
  administrators: number[];
  prevAdministrators: number[];
}
```

このプロパティをメソッド内の先頭で、以下のように値をセットし、変更前のユーザーを取得するようにしました。

```tsx
class Group {
  administrators: number[];
  prevAdministrators: number[];
  deleteInsert() {
    this.prevAdministrators = this.administrators;
    /** administratorsプロパティを書き換える処理 */
  }
}
```

これで、インターセプターでも既存の管理者情報が取得できるようになりました。
めでたしめでたし。
と言いたいのですが、先程管理者情報のエンドポイントは今後の結構編集されることがあるといいました。
それに付随して Group クラスのメソッドも増減し、その度にメソッド内の先頭へ`this.prevAdministrators = this.administrators;`を書く必要があります。
いつか絶対忘れてしまい、通知対象の人が適切に取得できなくなるケースが発生しますね。
それを防ぐための方法についてが、今回の記事となります。
カスタムデコレーターを作り、以下のようにクラスの上に付与するだけで、メソッドが実行する前に対象のプロパティへ、他のプロパティの値をセット処理を自動で走るようにします。

```tsx
@Hoge("hoge")
class Group {}
```

これによって、実装者に依存せず適切な管理者情報を取得できるようになります。
以上が今回記事を書く経緯となります。
それでは、実装について見ていきます。

# 実装内容の解説

## 全体的なコード

先に全体的なコードを展開します。

```tsx
function StorePrevValue<U extends { prototype: object }>(
  values: {
    prpropertyNameForSet: ExtractProperties<U["prototype"]>;
    prpropertyNameForGet: ExtractProperties<U["prototype"]>;
  }[]
) {
  return function <T extends { new (...args: any[]): {} }>(constructor: T) {
    /** クラスに存在するすべてのメソッド名を取得
     *  ただし、コンストラクタだけはプロパティの上書き処理をさせたくないので、除外
     */
    const methods = Object.getOwnPropertyNames(constructor.prototype).filter(
      (method) => method !== "constructor"
    );
    methods.forEach((methodName) => {
      const originalMethod = constructor.prototype[methodName];
      if (typeof originalMethod === "function") {
        /** 関数を上書きする */
        constructor.prototype[methodName] = function (
          this: any,
          ...args: any[]
        ) {
          /** メソッドが実行される前の値を指定のプロパティに格納 */
          values.forEach((val) => {
            this[val.prpropertyNameForSet] = this[val.prpropertyNameForGet];
          });
          /** 元のメソッドを呼び出し */
          return originalMethod.apply(this, args);
        };
      }
    });
    return constructor;
  };
}
type ExtractProperties<T> = {
  [K in keyof T]: T[K] extends Function ? never : K;
}[keyof T];
@StorePrevValue<typeof Group>([
  {
    prpropertyNameForSet: "prevAdministrators",
    prpropertyNameForGet: "administrators",
  },
])
class Group {
  administrators: number[];
  readonly prevAdministrators: number[] = [];
  constructor(administrators: number[]) {
    this.administrators = administrators;
  }
  setAdministrators(administrators: number[]) {
    this.administrators = administrators;
  }
}
const group = new Group([1, 2]);
group.setAdministrators([4, 5, 6]);
console.log(group.prevAdministrators, "prevAdministrators");
console.log(group.administrators, "administrators");
group.setAdministrators([7, 8, 9]);
console.log(group.prevAdministrators, "prevAdministrators_2");
console.log(group.administrators, "administrators_2");
```

プレイグラウンドのリンクも下記に添付します。
[プレイグラウンド](https://www.typescriptlang.org/play/?#code/FDBmFcDsGMBcEsD2kAEBlWiBOBTACrgG4BqAhgDbg4A8AqijgB6w6QAmAzigN4oAOWRJlgBPPjgBcKRACMAVjjgoAvgD4AFIQpUOU3gIGJxWUQDlSAWxwAxbGhywpAUWZZScAkZwn4ODnQBtAHJDYTEcIIBdVQAafixDYzNLG2wAcQdnV3dYTyTff1pg0KFwqNUVAMiASh5gFBRcWHAsVAgYBGQUagAVBmZWTh4USBwAd3UAOmnSLABzXRRSSBEq6r1lFQ1oZA5YLHA4bCke2u4QBoaAegAqG5RAeoZAS4ZAToZAawZADW1ACnVATQZAaIYfoBPBkAZgyAOwZAIcMgF6GQDDDIBJhkAsCqAJIZAGvKgHT9eqXFD3FCAfQZAAYMgHUGQCADIBmhkAzwwvQATDE8HoB+hjxgEUGQD2DIB1hkAtwyARYZAGMMgGKGUGAKDlAB9mgFkGQBnioAwF0RgFUGQDaDDjAPIMgCsGQAiDKDAOYMRMAJmmANE0MZcblc9SgdpA9igrLAABaIIYAXhQAHl5IpYJM5g4HWNIHlvMkrBx1Ca9gcjlhJiVROJqpNQPByCwsOoLda2ChbRVkzaUABCW32oJB-aHTBYILVADcF0umc4MewTnclvUSYcKfMVlq6bqmIahekWHgc3gkAoAFlW1n7YWQyXw4IwuIAjX2zhIpWjQ14KAUOpIzhENvsIPh2OJ6m8-n2nAkJAy92e9c7ihAEWpgAdTRFC4X-LGGh+93ZFqGc5CKUi7LikkRpigV6dKgu6WvAizLCIcTTJMswLFIyFrPef4oLc9xQnCgAyDIA+dqADIRUqADEMfyALJKoKACQKiKAODGgBZ2qC7LcjybyADwWgCwvj+G49lolB+HWWANtATYiWmFTnHhmJWohAQiXOiS+iIK62Fg9iwJB9pKRwKkUGpghJJpKTaRkelCZiyjVFWeEESggDCihCMKwoigA+KoAzgyAF+KBKCQpjQOC0qBHkOI7kOOVo2uhfB8OQIjwYhcQYRwFa2SolYPsoRr2eumJNGFxoATO2A5dlwB5cAe4oC4+w5D6Ph+L0FT2vJKABAA0igw4oAA1jgIgHigPSRCcPWQUwLDsFw1hQNeXQAPwjDghDeCgUjddVARDSN27jeuAACGDYPgRBkKJ1B7qNaSCOAfAaAEGL6AkZkaVpdiZCgIREAAgmwFjDohjUlhwQRxAYH0mBZVhWT9QSkEDIPBqQ4NBCoMTADUwDQOQpAcFw92II997I8DkCg244NSJA4AWDI3hVBiuDI8gSXxOtgOU9T6PYIs9OM8z+ldWuVbTsW2DqBTqNgwLdMM0zWA4Z1DSGehKNU2j4NQbL2vy1gHCVTVDQcA4PNyzTAsy1rfO0yMSsi2cRoa-r9sC3rds6wLJsgDVfZzA9fBQaMYwoCTj3qAEACMMQAEw1JWQek3wkzm7AlsG9bRvRwALDEACsMQAGy40GiDkDgkzkIgczqCnj1ztz3uGxwMR-S3vM+0bZZ47slfV7X9eN2n7s9+3SOtznEMOaP6cW9P-O5wEADsMQABwxAAnOXA9VzXdcN8HzeEFnHtGx3Ahd1by8cAA+vHfcVwfw-H6nmvd23Hfj23j990AA)
では、それぞれ着目する点を順にみていきます。

## クラスからプロパティを抽出する ExtractProperties

以下のコード部分です。

```tsx
type ExtractProperties<T> = {
  [K in keyof T]: T[K] extends Function ? never : K;
}[keyof T];
```

それぞれの処理を確認します。
まず、`[K in keyof T]`ですがこれはジェネリクス型である T のプロパティを keyof で抽出しています。
そして、in としているので、T に含まれるプロパティ全てをオブジェクト内に定義することが求められます。
例を出します。
以下のような型定義があります。

```tsx
type ObjectTest = {
  id: number;
  id2: string;
  id3: string;
};
type NewObject = {
  [K in keyof ObjectTest]: any;
};
```

この時、NewObject 型とする変数を定義します。

```tsx
const testObject: NewObject = {};
```

これに補完を効かしてみると以下のようになります。
![2024-08-03_17h12_56.png](/images/nestjs-custom-decolator/2024-08-03_17h12_56.png)
ObjectTest 型のプロパティが出てきますね。
変数 testObject は id,id2,id3 を定義しないと、型エラーが発生します。
ただ、値については NewObject 型では any 型となっているので、ObjectTest 型の値と合わせる必要はありません。
このようにプロパティに、`[K in keyof T]`といった形で定義することで、T のプロパティ全てを定義することを強制できます。
[プレイグラウンド](https://www.typescriptlang.org/play/?#code/LAKALgngDgpgBAeQEYCsYGMwBUYGcxwC8cA3qHBXAJYAmAXAHYCuAtkjAE4Dc5ltATHXwcqDAOY8QlajQDMQsCPGgAvqFCRYcAHIwA7sjSYipXhQDaAaWoM4AaxgQA9gDNEqDNjxgAunQCGDBCq6iDoTgz4cGDehp50ugYexsRkUnz0AIwANGYygvy56RS08rKqQA)
プロパティ部分は確認したので、次は値の型定義である`T[K] extends Function ? never : K`を見ます。
これは、T の中に存在するするプロパティ K の値が関数であれば、never 型とし、そうでなければ K を型定義とするようにしています。
ざっくり言ってしまえば、型定義の三項演算子を表しています。
例で確認していきます。
以下のクラスがあるとします。

```tsx
class SimpleClass {
  name: string = "";
  age: number = 1;
  greet() {
    console.log("Hello");
  }
}
```

このクラスを用いて、先程の型定義の`[keyof T]`以外の部分を当てはめてみます。

```tsx
/** neverという値はないのでエラーが発生する */
const test: {
  [K in keyof SimpleClass]: SimpleClass[K] extends Function ? never : K;
} = {
  age: "age",
  name: "name",
  greet: never,
};
```

age と name は関数ではないので、それぞれプロパティ名の文字列のみ定義できます。
あくまでプロパティ名である K が型となるので、任意の文字列を設定できないことはご注文ください。
そして、関数は never 型なのでプロパティ名を入力することはできません。
それどころか、実際上記のような greet プロパティは型が never なので設定すらできません。
なので、test 変数は定義できません。
あくまでイメージしてもらうために、記載しただけなのでご注意ください。
動くものとしては、以下の中間型を定義します。

```tsx
type SampleClassTypes = {
  [K in keyof SimpleClass]: SimpleClass[K] extends Function ? never : K;
};
```

これは先程見た変数の挙動からも分かるように、以下の型と同義です。

```tsx
type SampleClassTypes = {
  name: "name";
  age: "age";
  greet: never;
};
```

[プレイグラウンド](https://www.typescriptlang.org/play/?#code/FAYwNghgzlAEDKBLAtgBzAUwMKRrA3sLLAHYTIYBcsUALgE6IkDmsAvLAOScDcRsEZlVIBXZACMM9drACMfYs3oYMtABQBKArBAB7ElF2YAdGF3M1AIgASGMGcsaesAL7A3wAPQAqb6QwAblKAFgyAIgyAYgyAJAqA9gyAVgyhgHYMgOYMgBUMgJcMgD8MgDIMgF5ugPiugJoMgNEMsKD6dLC0GHSUhMQA2gDSsEywANYYAJ66AGYIKOjYuFAAutRIaJg40FBNI7AYAB5VJAAmcABiIiQgtIj6sAD8-kHS1I3uMnUCQpScghicADT8ZBR3b48visqqlCSBKR8NzeTzAYC0TqoDAIciDaYwAAqUOqV34TRaJHaXV6-UmQxmYzx8OGcwWywwa02212+yxxwBp1g53cXl8RD8gCg5QAYUYBo9WigFO5QDQcglABEM4UA1gyxIoc0qwPyQ6Gw-EIqDI6FwDh1PykcjCSyfSwKHUPaiWB5G2WwJQqWjURlA2UuY1g8FAA)
ここまでオブジェクト部分の型定義について見てきたので、最後に

```tsx
{
  [K in keyof T]: T[K] extends Function ? never : K
}[keyof T]
```

の`[keyof T]`を絡めた部分の挙動を確認します。
なお、ここは自分が分かりやすい理解で記載しているため、実際の挙動の説明になっているかは担保できないので、ご注意ください。
`keyof T`はオブジェクトのプロパティのユニオン型を作ります。
例えば、以下の型定義があるとします。

```tsx
type TestObject = {
  id: string;
  name: string;
};
```

この TestObject 型に対して keyof をすると以下の関係が成り立ちます。

```tsx
type TestObjectKeys = keyof TestObject = 'id' | 'name'
```

プロパティの名前しか入力できなくなります。
試しに TestObjectKeys 型を使った変数を定義すると、以下のように文字列の補完がでます。
![2024-08-03_21h39_39.png](/images/nestjs-custom-decolator/2024-08-03_21h39_39.png)
変数 key には id か name という文字列しか設定できず、それ以外はエラーが発生します。
このようにオブジェクトを keyof すると、プロパティのユニオン型を作成します。
[プレイグラウンド](https://www.typescriptlang.org/play/?#code/C4TwDgpgBAKhDOwDyAjAVhAxsKBeKA3gFBRQCWAJgFwB2ArgLYoQBOA3CVDQIYMRWIWZGgHMOAXyJFQkWAmToswANIQQ8PFADWagPYAzOYlQZsUzLpqJtaqnGOLsq9ZoDklV0A)
ではこのことを踏まえて、`{}[keyof T]`はどのような動きをするか確認します。
`keyof T`はユニオン型を作り、それを鍵括弧で囲っています。
この鍵括弧は今回のケースにおいて、配列ではなくオブジェクトのプロパティアクセスとなります。
イメージは以下のような感じです。

```tsx
const testObj = {
  id: 1,
};
const id = testObj["id"];
```

これらのことから、`{}[keyof T]`はオブジェクトに対して各プロパティに対応する型のユニオン型となります。
例を、先程作った TestObject を用いて確認します。
本来の型定義にある程度近い ExtractProperties 型を以下のように定義します。

```tsx
type TestObject = {
  id: number;
  name: string;
};
type ExtractProperties = {
  [K in keyof TestObject]: any;
}[keyof TestObject];
```

ExtractProperties 型は以下の型と同義になります。

```tsx
type ExtractProperties =
  | { id: any; name: any }["id"]
  | { id: any; name: any }["name"];
```

そして、型の時も同様にプロパティを指定したオブジェクトへのアクセスの場合、一致するプロパティの value に設定している型を返します。
つまり、先程の ExtractProperties は以下のように解釈されます。

```tsx
type ExtractProperties = any | any;
```

どちらも any 型なので、例の型の場合`type ExtractProperties = any`となります。
次に、本来の型定義へより寄せていきます。
説明したいことの都合上、以下 TestObject2 型を作ります。

```tsx
type TestObject2 = {
  id: number;
  name: string;
  func: () => void;
};
```

この型をもとに ExtractProperties2 を設定します。

```tsx
type ExtractProperties2 = {
  [K in keyof TestObject2]: TestObject2[K] extends Function ? never : K;
}[keyof TestObject2];
```

先程の説明から、ExtractProperties2 型は以下と同義になります。

```tsx
type ExtractProperties2 = "id" | "name" | never;
```

ただし、ユニオン型において never を定義しても無視されます。
つまり、ExtractProperties2 型は以下のように解釈されます。

```tsx
type ExtractProperties2 = "id" | "name";
```

[プレイグラウンド](https://www.typescriptlang.org/play/?#code/C4TwDgpgBAKhDOwDyAjAVhAxsKBeKA3gFBRQCWAJgFwB2ArgLYoQBOA3CVDQIYMRWIWZGgHMOAXyJFQkKAFEAHsBbdsABRYB7SC2BkEeQpwDaAaXI0oAawghNAM1gJk6LMAC6VKNxogi44xs7RzhEVAxsdw4iAHoAKjioEkTAU7lAaDlACwZAGBVASv9kqBloRWVVYA1tVj0DfAJyam9fABouXn4GkCgAgHJKLvcoAB9COq8fEGaePlHfTuMuyYg+jihEwBMGQD8GQCiGQEAGQvklFXUtHSr4QzHB9sArBkA7BkBzBm38veLDsuPK-TPcQAAGMYyrwDRDIAgBmSMSk0nA0FCLgiwAATIZiKRKLRGMx2JwFgJlMIxJx7HQaJgqAAKACUuAAfAA3TSUCRSZ4HUrlE6fBE1EzmYTWWwOJxhVzYOGeAUwtxwsz9CBKCA0ChnABihOwZE0lgA-FwINTWFAvKYJIE+SFnOEJVEgA)
ようやく全貌が見えてきました。
以上のことから、任意のクラスにおいてメソッドを省きプロパティの名前のみを抽出するには以下コードで動く理由が分かりました。

```tsx
type ExtractProperties<T> = {
  [K in keyof T]: T[K] extends Function ? never : K;
}[keyof T];
```

ただ、これは前座です。
次に本丸である、デコレーターの定義をみていきます。

## 指定のプロパティに値をセットするデコレーター

以下の部分のコードになります。

```tsx
function StorePrevValue<U extends { prototype: object }>(
  values: {
    prpropertyNameForSet: ExtractProperties<U["prototype"]>;
    prpropertyNameForGet: ExtractProperties<U["prototype"]>;
  }[]
) {
  return function <T extends { new (...args: any[]): {} }>(constructor: T) {
    /** クラスに存在するすべてのメソッド名を取得
     *  ただし、コンストラクタだけはプロパティの上書き処理をさせたくないので、除外
     */
    const methods = Object.getOwnPropertyNames(constructor.prototype).filter(
      (method) => method !== "constructor"
    );
    methods.forEach((methodName) => {
      const originalMethod = constructor.prototype[methodName];
      if (typeof originalMethod === "function") {
        /** 関数を上書きする */
        constructor.prototype[methodName] = function (
          this: any,
          ...args: any[]
        ) {
          /** メソッドが実行される前の値を指定のプロパティに格納 */
          values.forEach((val) => {
            this[val.prpropertyNameForSet] = this[val.prpropertyNameForGet];
          });
          /** 元のメソッドを呼び出し */
          return originalMethod.apply(this, args);
        };
      }
    });
    return constructor;
  };
}
```

この部分の動きについて確認する前に、まず Typescript におけるデコレーターとは何かをみていきます。

### デコレーターとは

デコレーターとは[typescript のアナウンス](https://devblogs.microsoft.com/typescript/announcing-typescript-5-0/#decorators)では以下のように記載されています。

> Decorators are an upcoming ECMAScript feature that allow us to customize classes and their members in a reusable way.

クラスや内部のメソッドを再利用可能な形でカスタマイズする時に活用できるとしています。
コードとしては、NestJS で開発しているとよく`@`から始まる単語をメソッドやクラスの上とかに書くと思います。
あれがデコレーターの呼び出し方になります。
とりあえずカスタマイズする時に役立ち、`@`から始めればデコレーターとして機能することは分かりました。
ただ、このままではなぜカスタマイズが効くのかがよくわかりません。
なので、デコレーターがどのような処理になるかをみていきます。
まず、前提としてデコレーターは関数です。
[Typescript のアナウンス](https://devblogs.microsoft.com/typescript/announcing-typescript-5-0/#decorators)を見てみると、デコレーターの元となる処理を以下のように記載しています。

```tsx
function loggedMethod(originalMethod: any, _context: any) {
  function replacementMethod(this: any, ...args: any[]) {
    console.log("LOG: Entering method.");
    const result = originalMethod.call(this, ...args);
    console.log("LOG: Exiting method.");
    return result;
  }
  return replacementMethod;
}
```

そして、それを以下のように設定するだけで、greet メソッド実行時に loggedMethod 関数が実行されます。

```tsx
class Person {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
  @loggedMethod
  greet() {
    console.log(`Hello, my name is ${this.name}.`);
  }
}
```

実際に[プレイグラウンド](https://www.typescriptlang.org/play/?#code/GYVwdgxgLglg9mABAGzgczQUwCYFlNQAWc2AFHAE4xoxgCGy+RJAXInWAJ4A0iA+hARRMADyhsOnAJSIA3gCh5iZYlCRYCRBUwAHZHQiYAtpjBQmxMkRgBnCV14A6Z3Qpo77LgG0AujIUqgYiCYDZwyJiOqGikAEQAMgDyAOJsAKJmmFRgaIgmzNiOsVJKQcohNlBamDYgyFUAvIiU1LQMFiSOEAzIpNY2Ti5uNlIA3KVlFeGR0XFJqYhpIjCwOXkElkUlZcraUCAUSNq19eOBAL6KgXsHR7r6hiZmHdjjl4oQ+jY2iAAKWWEkAEVPQTGxKtk0GcVBUoBQQNBKKRQZhwXDaGh-BMgv1HCjEE0UdDlO9AgABaJYPAbEjYtDaAikLE7YIIMIRKLoUgAAwAEphkKheEZOIh8bZEAASWS4lHnRzcsbYy7vWGIHQEsWYADufwBCDiACUEMVxjpHPTMIyxkA)で確認してみると、loggedMethod が呼び出されていることが確認できます。
ただ、プレイグラウンドで確認すると少し不思議な挙動をしていると思います。
それは、ログが以下のように出力されるためです。

```tsx
[LOG]: "LOG: Entering method."
[LOG]: "Hello, my name is Ron."
[LOG]: "LOG: Exiting method."
```

loggedMethod 関数は greet を実行する前に呼ばれているので、何となく"Hello, my name is Ron." が最後に出力されそうです。
しかし、実際は"Hello, my name is Ron." が真ん中に来ています。
これは、loggedMethod 関数が greet メソッドの書き換えを行っているためです。
デコレーター用の関数は引数に必ず、実行されるメソッドとメソッドの種類についての情報設定されます。
実行されるメソッドは第一引数で、メソッドの情報は第二引数です。
このことを踏まえて再度デコレーター用の関数のコードを見てみます。

```tsx
function loggedMethod(originalMethod: any, _context: any) {
  function replacementMethod(this: any, ...args: any[]) {
    console.log("LOG: Entering method.");
    const result = originalMethod.call(this, ...args);
    console.log("LOG: Exiting method.");
    return result;
  }
  return replacementMethod;
}
```

replacementMethod 関数を新たに定義し、それを返しています。
replacementMethod 関数の中では、console.log を加え、originalMethod を call しています。
originalMethod はメソッドが入ってきているので、今回でいうと greet メソッドが該当します。
それを call していることから、元々の greet メソッドが実行されることが分かります。※
ただ、`person.greet()`といった形で呼び出すと loggedMethod 関数の戻り値である、replacementMethod 関数の処理が呼び出されます。
よって、デコレーター用の関数は元のメソッドを活用しつつ、元々のメソッドを上書きする関数だといえそうです。
※call など prototype が持つメソッドについては私がちゃんと理解できていないので、第一引数の this などの解説をしておりません。ここについては、[ドキュメント](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Function/call)などを参照していただけますと幸いです。
以上がデコレーターの挙動についての確認でした。
カスタマイズできるというのは、デコレーターが元のメソッドを受け取りつつ、メソッドを上書きできるからなんですね。
試してみると確かにデコレーターはカスタマイズしやすい仕組みだなと感じます。
なお、今はメソッドへのデコレーターを中心に見ていきましたが、以下のようにクラスに対してデコレーター関数を設定すると、第一引数にコンストラクタ関数が入ってきます。

```tsx
@Hoge
class Piyo {}
```

私自身コンストラクタ関数の理解をきちんとできていないので、間違っている可能性はありますが、デコレーターを設定したクラスの情報が入ってきていると認識して、実装すれば問題はなさそうでした。
今回実装したデコレーター関数はクラス用なので、引数としてはこちらが設定されます。
ここまでデコレーターについて確認してきたので、いよいよ今回のデコレーター関数の中身を解説していきます。

### デコレーター関数 StorePrevValue が何をしているのかを見る

まず StorePrevValue 関数を再掲します。

```tsx
function StorePrevValue<U extends { prototype: object }>(
  values: {
    prpropertyNameForSet: ExtractProperties<U["prototype"]>;
    prpropertyNameForGet: ExtractProperties<U["prototype"]>;
  }[]
) {
  return function <T extends { new (...args: any[]): {} }>(constructor: T) {
    /** クラスに存在するすべてのメソッド名を取得
     *  ただし、コンストラクタだけはプロパティの上書き処理をさせたくないので、除外
     */
    const methods = Object.getOwnPropertyNames(constructor.prototype).filter(
      (method) => method !== "constructor"
    );
    methods.forEach((methodName) => {
      const originalMethod = constructor.prototype[methodName];
      if (typeof originalMethod === "function") {
        /** 関数を上書きする */
        constructor.prototype[methodName] = function (
          this: any,
          ...args: any[]
        ) {
          /** メソッドが実行される前の値を指定のプロパティに格納 */
          values.forEach((val) => {
            this[val.prpropertyNameForSet] = this[val.prpropertyNameForGet];
          });
          /** 元のメソッドを呼び出し */
          return originalMethod.apply(this, args);
        };
      }
    });
    return constructor;
  };
}
```

順にみていきます。
最初に、StorePrevValue 関数の定義です。

```tsx
StorePrevValue<U extends { prototype: object }>(values: { prpropertyNameForSet: ExtractProperties<U['prototype']>, prpropertyNameForGet: ExtractProperties<U['prototype']> }[])
```

ジェネリクス型 U にクラスオブジェクトがくることを想定しています。
そのために、prototype をプロパティに持つオブジェクトの拡張型であること明示的にしています。
これは、引数にそれぞれメソッドを実行する前に既存の値を保持しておくプロパティと、値を保持するプロパティに渡す値を持つプロパティを補完が効く形で指定したかったためです。
なぜ補完が効くかというと、先程解説した ExtractProperties 型を引数の型として指定しているためです。
ExtractProperties 型を使うことで、クラスの prototype においてプロパティだけをユニオン型で抽出できるので、値の打ち間違いを防ぐことができます。
なお、先程クラスのデコレータは第一引数にコンストラクタ関数を設定するといいました。
ですが、ここでは第一引数にコンストラクタ関数がくることを想定していません。
これでもちゃんと動きます。
ただし、クラスにデコレートするときは`@StorePrevValue`という形ではなく、`@StorePrevValue([])`といった形で一回関数を実行する必要があります。
これによって、内部で定義した以下関数部分がクラスのデコレーターとして設定されるためです。

```tsx
return function <T extends { new (...args: any[]): {} }>(constructor: T) {
  /** 省略 */
  return constructor;
};
```

なので、関数を定義する時は必ず第一引数にコンストラクタ関数を設定しなくてもいいですが、クラスのデコレーターとして設定するときは第一引数がコンストラクタ関数となっている関数を渡す必要があります。
今回は戻り値にコンストラクタ関数を受け取る関数を設定しているので、問題ありません。
では、内部の処理をみていきます。

```tsx
/** クラスに存在するすべてのメソッド名を取得
 *  ただし、コンストラクタだけはプロパティの上書き処理をさせたくないので、除外
 */
const methods = Object.getOwnPropertyNames(constructor.prototype).filter(
  (method) => method !== "constructor"
);
methods.forEach((methodName) => {
  const originalMethod = constructor.prototype[methodName];
  if (typeof originalMethod === "function") {
    /** 関数を上書きする */
    constructor.prototype[methodName] = function (this: any, ...args: any[]) {
      /** メソッドが実行される前の値を指定のプロパティに格納 */
      values.forEach((val) => {
        this[val.prpropertyNameForSet] = this[val.prpropertyNameForGet];
      });
      /** 元のメソッドを呼び出し */
      return originalMethod.apply(this, args);
    };
  }
});
return constructor;
```

コード内に書いてあるコメント通りですが、やりたかったのは

```tsx
constructor.prototype[methodName] = function (this: any, ...args: any[]) {};
```

で関数を上書くことと、

```tsx
values.forEach((val) => {
  this[val.prpropertyNameForSet] = this[val.prpropertyNameForGet];
});
```

で指定したプロパティに値をセットすることです。
そして、その後メソッドを実行するようにしています。
デコレーター関数も定義できたので、最後に以下のようにクラスに対して設定します。

```tsx
@StorePrevValue<typeof Group>([
  {
    prpropertyNameForSet: "prevAdministrators",
    prpropertyNameForGet: "administrators",
  },
])
class Group {
  administrators: number[];
  readonly prevAdministrators: number[] = [];
  /** 省略 */
}
```

使い方としては、まず StorePrevValue 関数の<>部分に実行させたいクラスの型を、typeof を使い記載します。
すると引数で設定する値は画像のように補完が効くようになります。
![2024-08-04_17h22_27.png](/images/nestjs-custom-decolator/2024-08-04_17h22_27.png)
後はメソッド実行前に前の値を保持しておきたいプロパティと、保持しておく値を持つプロパティを指定すれば、メソッドを実行する前に指定のプロパティに値がセットされていることを確認できます。
挙動について、[プレイグラウンド](https://www.typescriptlang.org/play/?#code/FDBmFcDsGMBcEsD2kAEBlWiBOBTACrgG4BqAhgDbg4A8AqijgB6w6QAmAzigN4oAOWRJlgBPPjgBcKRACMAVjjgoAvgD4AFIQpUOU3gIGJxWUQDlSAWxwAxbGhywpAUWZZScAkZwn4ODnQBtAHJDYTEcIIBdVQAafixDYzNLG2wAcQdnV3dYTyTff1pg0KFwqNUVAMiASh5gFBRcWHAsVAgYBGQUagAVBmZWTh4USBwAd3UAOmnSLABzXRRSSBEq6r1lFQ1oZA5YLHA4bCke2u4QBoaAegAqG5RAeoZAS4ZAToZAawZADW1ACnVATQZAaIYfoBPBkAZgyAOwZAIcMgF6GQDDDIBJhkAsCqAJIZAGvKgHT9eqXFD3FCAfQZAAYMgHUGQCADIBmhkAzwwvQATDE8HoB+hjxgEUGQD2DIB1hkAtwyARYZAGMMgGKGUGAKDlAB9mgFkGQBnioAwF0RgFUGQDaDDjAPIMgCsGQAiDKDAOYMRMAJmmANE0MZcblc9SgdpA9igrLAABaIIYAXhQAHl5IpYJM5g4HWNIHlvMkrBx1Ca9gcjlhJiVROJqpNQPByCwsOoLda2ChbRVkzaUABCW32oJB-aHTBYILVADcF0umc4MewTnclvUSYcKfMVlq6bqmIahekWHgc3gkAoAFlW1n7YWQyXw4IwuIAjX2zhIpWjQ14KAUOpIzhENvsIPh2OJ6m8-n2nAkJAy92e9c7ihAEWpgAdTRFC4X-LGGh+93ZFqGc5CKUi7LikkRpigV6dKgu6WvAizLCIcTTJMswLFIyFrPef4oLc9xQnCgAyDIA+dqADIRUqADEMfyALJKoKACQKiKAODGgBZ2qC7LcjybyADwWgCwvj+G49lolB+HWWANtATYiWmFTnHhmJWohAQiXOiS+iIK62Fg9iwJB9pKRwKkUGpghJJpKTaRkelCZiyjVFWeEESggDCihCMKwoigA+KoAzgyAF+KBKCQpjQOC0qBHkOI7kOOVo2uhfB8OQIjwYhcQYRwFa2SolYPsoRr2eumJNGFxoATO2A5dlwB5cAe4oC4+w5D6Ph+L0FT2vJKABAA0igw4oAA1jgIgHigPSRCcPWQUwLDsFw1hQNeXQAPwjDghDeCgUjddVARDSN27jeuAACGDYPgRBkKJ1B7qNaSCOAfAaAEGL6AkZkaVpdiZCgIREAAgmwFjDohjUlhwQRxAYH0mBZVhWT9QSkEDIPBqQ4NBCoMTADUwDQOQpAcFw92II997I8DkCg244NSJA4AWDI3hVBiuDI8gSXxOtgOU9T6PYIs9OM8z+ldWuVbTsW2DqBTqNgwLdMM0zWA4Z1DSGehKNU2j4NQbL2vy1gHCVTVDQcA4PNyzTAsy1rfO0yMSsi2cRoa-r9sC3rds6wLJsgDVfZzA9fBQaMYwoCTj3qAEACMMQAEw1JWQek3wkzm7AlsG9bRvRwALDEACsMQAGy40GiDkDgkzkIgczqCnj1ztz3uGxwMR-S3vM+0bZZ47slfV7X9eN2n7s9+3SOtznEMOaP6cW9P-O5wEADsMQABwxAAnOXA9VzXdcN8HzeEFnHtGx3Ahd1by8cAA+vHfcVwfw-H6nmvd23Hfj23j990AA)を確認ください。
なお、保持しておきたいプロパティはクラス内で更新する処理を実装させたくないので、 readonly を付与するのを推奨します。
readonly を付与すれば、クラス内で値の置き換えができないので、予期せぬ値にならないかなと思います。

# おわりに

今回はクラス内のメソッドを実行する時に、値が書き変わる前のプロパティを保存する方法についての解説でした。
やりたいことの大きさの割に、理解しておくことが多く大変でした。
ただ、そのおかげで何となく書いていたデコレーターの挙動とその効果を実感できました。
乱用するとそれはそれで読みにくいコードになってしまいますが、活用できる場面では活かしていきたいと思います。
ここまで読んでいただきありがとうございました。
