---
title: "ULIDのタイムスタンプとソートをちゃんとわかっていなかった話"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ULID", "Sort", "Timestamp"]
published: true
---

## はじめに

タイムスタンプをもとにした文字列を付与することで、ソート可能となり、さらに最終的には一意のランダムな文字列を作成してくれる ULID。
私は上記のような理解をしていました。
そのため、とりあえず ULID で採番しておけばソートできると思っていました。
しかし、その理解は部分的には誤っていました。
それを確認するために、以下のコードを見てください。

```jsx
import { ulid } from 'ulid'
const ids = [1, 2, 3, 4, 5, 6, 7].map(v => ({ id: v, ulid: uli() }))
console.log(ids.sort((a, b) => {
    if (a.ulid > b.ulid) {
        return -1
    }
    if (a.ulid < b.ulid) {
        return 1
    }
    return 0;
})
```

`ulid()`は[ulid パッケージ](https://github.com/ulid/javascript)による提供で、ULID を生成する関数です。
これを値とした、配列を作っています。
では、これを実行してみてください。
元々の私の ULID への理解では、id の値が 7 となるオブジェクトが配列の最初になり、以降は 6,5,4…と降順となります。
ですが、実際に 3 回ほど実行するとそれぞれ以下のようになります。

```jsx
/** 実行1回目 */
[
  { id: 3, ulid: "01J6VWSHZJYTCEMWNQ9V5VXQ80" },
  { id: 6, ulid: "01J6VWSHZJMBRT6225XZ7J5WR4" },
  { id: 5, ulid: "01J6VWSHZJG8TDKD2SXD64QZV5" },
  { id: 7, ulid: "01J6VWSHZJ90VFMF45DPHJBCRD" },
  { id: 4, ulid: "01J6VWSHZJ31QSMCV7MSSTJ3QG" },
  { id: 2, ulid: "01J6VWSHZHW2D3PVCSGX4W211X" },
  { id: 1, ulid: "01J6VWSHZH0X8MB18EEC1BH2M3" },
][
  /** 実行2回目 */
  ({ id: 2, ulid: "01J6VWSE7DXAA35D718XHGFKTH" },
  { id: 3, ulid: "01J6VWSE7DM4HRXBZ6X5DNFMGJ" },
  { id: 7, ulid: "01J6VWSE7DBDTYFXJ6SYTRJ8T3" },
  { id: 6, ulid: "01J6VWSE7D9B0CJGMGMJA5391Z" },
  { id: 4, ulid: "01J6VWSE7D6N4985P1XWA5SKVX" },
  { id: 5, ulid: "01J6VWSE7D4Q29SY2QMV4SA4TX" },
  { id: 1, ulid: "01J6VWSE7CG76MK61F2689J96W" })
][
  /** 実行3回目 */
  ({ id: 7, ulid: "01J6VWS9NSGWKWD7Q0GEN69T23" },
  { id: 6, ulid: "01J6VWS9NS8YMWN7Y16GR881H2" },
  { id: 3, ulid: "01J6VWS9NRSR0S97EDXGJ03ZCN" },
  { id: 4, ulid: "01J6VWS9NRQ8SCWJHDAW2HTNWB" },
  { id: 2, ulid: "01J6VWS9NRQ7M81GXMJDSQ9Z0W" },
  { id: 5, ulid: "01J6VWS9NR6C43JR6TZJYME1YB" },
  { id: 1, ulid: "01J6VWS9NQNFWKRC5RMT4S2HT2" })
];
```

全然順番が安定しません。
これは ULID はタイムスタンプを使っているので、ソート可能という話から考えると、おかしいように感じます。
ですが、これは正常動作です。
今回はなぜこの ULID の挙動は正常動作なのかを見ていき、とはいえサンプルコードのようなケースでもソートしたい時の方法について見ていきます。

## まず ULID が何かを確認する

ULID の動作が正しいかを判断するには、ULID が何者かを確認する必要があります。
ULID は[ドキュメント](https://github.com/ulid/spec?tab=readme-ov-file#specification)を確認すると以下のように構成されています。

```jsx
 01AN4Z07BY      79KA1307SR9X4MV3
|----------|    |----------------|
 Timestamp          Randomness
   48bits             80bits
```

上記の構造の説明として、以下の記載があります。

> **Components**
>
> **Timestamp**
>
> - 48 bit integer
> - UNIX-time in milliseconds
> - Won't run out of space 'til the year 10889 AD.
>
> **Randomness**
>
> - 80 bits
> - Cryptographically secure source of randomness, if possible

先頭の 48 ビットは ULID が生成された時のミリ秒での UNIX 時間を、表現しています。
そして、残りは可能な限り暗号的に安全なランダムな文字列で形成されています。
UNIX 時間については、上記にもあるように 10889 年までは適切に動作することを保証しています。
そして、この後に関わってくるので強調しておきますが、ULID は**ミリ秒**の UNIX 時間をエンコードしたものを出力しています。
ULID の構造は記載したので、ULID の特徴についても記載します。
が、この記事の主題とは直接関係する部分が少ないので、読み飛ばしても問題ありません。
再度[ドキュメント](https://github.com/ulid/spec)を確認すると、ULID の特徴は以下の通りです。

> - 128-bit compatibility with UUID
> - 1.21e+24 unique ULIDs per millisecond
> - Lexicographically sortable!
> - Canonically encoded as a 26 character string, as opposed to the 36 character UUID
> - Uses Crockford's base32 for better efficiency and readability (5 bits per character)
> - Case insensitive
> - No special characters (URL safe)
> - Monotonic sort order (correctly detects and handles the same millisecond)

UUID が 36 文字なのに対し、ULID は 26 文字で表現されます。
ですが、どちらもデータ量は 128 ビットなので、互換性を保っています。
1 ミリ秒あたり 1.21E+24(≒1.21x10^24)までの生成であれば、一意性を担保できます。
1 文字あたり 5 ビットである Base 32 を用いてエンコードしています。
構造の説明部分で、48 ビットなのに 10 文字となっていたのは、Base32 でエンコードしていたことが理由になります。
最後に、ULID は辞書的なソートができます。
これは、ULID の先頭 10 文字が生成されたときの UNIX 時間をもとになっており、生成時間に対応して文字列を生成しているためです。
ULID がソートできるのは、この UNIX 時間でのソート順となるように設計されているためです。
このタイムスタンプ通りにソートできるということだけは、私が把握していた通りでした。
以上が ULID の概要となります。

### 余談 ULID のライブラリでどうやってソートしているかを確認する

先程 ULID は先頭 10 文字が UNIX 時間をもとに作成した文字列なので、ソートができると言いました。
では、世にあるライブラリは上記のソース可能という要件を、どのように満たしているかを確認してみます。
今回見ていくのは、[ulid/javascript](https://github.com/ulid/javascript)です。
その中にある、UNIX 時間を文字列変換する[encodeTime 関数](https://github.com/ulid/javascript/blob/master/lib/index.ts#L66)を確認します。
ただし、コードをそのまま持ってくるのではなく、理解しやすいように一部改変はしています。
具体的には以下のコードです。

```jsx
const ENCODING = "0123456789ABCDEFGHJKMNPQRSTVWXYZ";
export function encodeTime(now: number): string {
  let mod;
  let str = "";
  for (let i = 10; i > 0; i--) {
    /**
     * 余りが必ず0~31になるようにする
     * Base32の文字列を取得できるようにするため
     */
    mod = now % 32;
    /** 一致する文字列を取得する */
    str = ENCODING.charAt(mod) + str;
    /** 余りを無くした状態で再代入 */
    now = (now - mod) / 32;
  }
  return str;
}
```

引数の数値を 32 で割り、そのあまりを取得します。
余りの値から、順番に一致する文字列を取得し、代入します。
文字列を取得したら、元の数字から余り分を引き 32 で割ります。
すなわち、32 進数基準で下一桁に合致する文字列を取得していることになります。
そして、取得する文字列は指定する順番が大きいほど、ソートとしては後に来るものになるように設定されています。
さらに、UNIX 時間は未来であればあるほど、値は必ず大きくなります。
以上のことから、ULID はタイムスタンプ通りにソートできるように作成していることが分かります。
UNIX 時間は常に増加し続ける数字であることと、32 で割り続けることで必ず値が大きい方が最終的な mod 値は大きくなるためです。
試しに、以下関数を好きな値を二つ入れてやってみてください。
必ず値が大きい方数を設定した方が、右から見た時大きい値で終わっています。

```jsx
function hoge(num: number) {
  let localNum = num;
  let str = "";
  for (let i = 0; i < 10; i++) {
    const mod = localNum % 32;
    str = `${mod}` + str;
    localNum = (localNum - mod) / 32;
  }
  console.log(str);
}
```

[プレイグラウンド](https://www.typescriptlang.org/play/?#code/GYVwdgxgLglg9mABACzgcwKYAowgLYBciueARhgE4CUiA3gFCKIA2GULcEAhswHL6IAvMXwBuRizaIAzlApDEAIkUTgceVlbsYCgAyidAHkQBGfTADUFmgyZMICWYjxwAJguace-PIgCkiADMAEwSTLLywgAGACS0Lq4AvlGIFjJyYRzcfALCml45vgC0zm40APRBoUyJEg5g0nCsAHSeaFgRVPS19KiYWIGBXX3YJsEAbFRAA)も置いておくので、試しみてください。
ULID はこの数値が大きいということを、辞書的に後に来る文字列に置き換えています。
よって、ULID は文字列ですがタイムスタンプでのソートができるようになります。

## ULID でソートが上手く行かなかった理由について

これまでで、ULID の概要を見てきました。
なので、その概要を踏まえてなぜ「はじめに」でのコードでは、上手くソートが出来なかったのかを確かめていきます。
ただ、これまで強調してきたので、ある程度推測はついているかもしれません。
そうです、ULID が参考にするタイムスタンプがミリ秒なのが理由です。
タイムスタンプがミリ秒基準なので、ミリ秒より早く処理が進んでしまう場合先頭 10 文字は同じ値になります。
そのため、ソートをする対象になってくるのが残りの文字列となります。
しかし、残りの文字列はランダムに生成されるので、ソート順を担保してくれません。
よって、ソートが上手く行かなかったのです。
ちなみに、ミリ秒より早く進むケースは簡単に再現できます。
for 文など、ループ処理を行うことです。
例えば、以下のコードを実行してみると、時より出力される値が同じになります。

```jsx
for (let i = 0; i < 5; i++) {
  console.log(Date.now());
}
/** 出力値 */
// 1725971252215
// 1725971252217
// 1725971252219
// 1725971252220
// 1725971252220
```

最後の二つが同じ値になっていますね。
このようにループ処理では、同じミリ秒で処理が完了してしまう時があります。
先程見た[ulid/javascript](https://github.com/ulid/javascript)の factory 簡単のコードをみると、定義漏れず`Date.now()`でミリ秒の値を出力するようにデフォルトでしています。

```jsx
export function factory(currPrng?: PRNG): ULID {
  if (!currPrng) {
    currPrng = detectPrng();
  }
  return function ulid(seedTime?: number): string {
    if (isNaN(seedTime)) {
      seedTime = Date.now();
    }
    return encodeTime(seedTime, TIME_LEN) + encodeRandom(RANDOM_LEN, currPrng);
  };
}
```

同じ値になってしまうため、後ろの文字列でソートを判断するが、後ろの文字列はソートを担保していないのです。
以上が、最初のコードでソートが上手く行かなかった理由です。
ちゃんと**ミリ秒**での文字列生成だと把握していればなんということのない話でしたが、とりあえずタイムスタンプを使っているからソートできると思っていると、今回みたいな落とし穴にはまります。

## とはいっても同一ミリ秒でソートしたい場合

ULID を使用すると、同一ミリ秒での生成ではソートが上手くいかないことを確認してきました。
定義から、確認するとその理由も納得できます。
ただ、場合によっては同一ミリ秒での生成でも、ソートを効かせたいと思う場合は存在すると思います。
ULID 側も上記ケースは想定しているので、ドキュメントに対応方法が記載してあります。
項目としては[Monotonicity](https://github.com/ulid/spec?tab=readme-ov-file#monotonicity)の部分です。
以下のように記載されています。

> When generating a ULID within the same millisecond, we can provide some guarantees regarding sort order. Namely, if the same millisecond is detected, the `random` component is incremented by 1 bit in the least significant bit position (with carrying).

ざっくり言うと、同一ミリ秒でのソート順を担保できると書いています。
具体的には、同じミリ秒であれば、最後に生成されたランダムな文字列部分を、1 ビット増加させる形で生成すると言及しています。
コードで見たほうがイメージ湧きやすいので、[ulid/javascript](https://github.com/ulid/javascript?tab=readme-ov-file#monotonic-ulids)で提供されている、Monotonicity な生成関数のサンプルコードを確認します。

```jsx
import { monotonicFactory } from "ulid";
const ulid = monotonicFactory();
// Strict ordering for the same timestamp, by incrementing the least-significant random bit by 1
ulid(150000); // 000XAL6S41ACTAV9WEVGEMMVR8
ulid(150000); // 000XAL6S41ACTAV9WEVGEMMVR9
ulid(150000); // 000XAL6S41ACTAV9WEVGEMMVRA
ulid(150000); // 000XAL6S41ACTAV9WEVGEMMVRB
ulid(150000); // 000XAL6S41ACTAV9WEVGEMMVRC
// Even if a lower timestamp is passed (or generated), it will preserve sort order
ulid(100000); // 000XAL6S41ACTAV9WEVGEMMVRD
```

同じミリ秒で生成した場合、前に作成した ULID の後ろの末尾部分が Base32 における、一つ後ろの文字列になっていますね。
後は、同じ文字列であることが確認できると思います。
このような形で、同一ミリ秒でもソートできる ULID を作りたい場合は、Monotonicity にして生成すると要件を達成できます。
ただ、注意点として、以前よりも ULID にパターンが生まれることになり、推測は以前よりしやすくなります。
そのため、重要な情報を取り出すためのキーにするのは、より適さなくなると思いますので、使用する状況によって、使い分ける必要はあります。
また、この方法は[ドキュメント](https://github.com/ulid/spec?tab=readme-ov-file#monotonicity:~:text=If%2C%20in%20the%20extremely%20unlikely%20event%20that%2C%20you%20manage%20to%20generate%20more%20than%20280%20ULIDs%20within%20the%20same%20millisecond%2C%20or%20cause%20the%20random%20component%20to%20overflow%20with%20less%2C%20the%20generation%20will%20fail.)に以下記載がある通り、1 ミリ秒間で 2^80 個以内の生成でのみソートが担保されます。

> If, in the extremely unlikely event that, you manage to generate more than 280 ULIDs within the same millisecond, or cause the random component to overflow with less, the generation will fail.

それ以上の時はエラーが発生しますので、超高速で処理を実行する場合はご注意ください。
ただ、正直アプリケーションで上記の限界に到達する状況はあまり思い浮かびませんが…
最後にもう一つ、ULID そのものというよりはライブラリについての注意点があります。
先程から紹介している、[ulid/javascript](https://github.com/ulid/javascript/tree/master)の monotonicFactory 関数ですが、タイムスタンプの引数を設定しない場合、あらかじめ monotonicFactory 関数を実行した状態でないとソートが担保されません。
コードでいうと、以下の使い方をするとソートが担保された状態での生成ができません。

```jsx
import { monotonicFactory } from "ulid";
for (let i = 0; i < 10; i++) {
  console.log(monotonicFactory()());
}
```

これの理由は、monotonicFactory 関数のコードを確認すると分かります。

```jsx
export function monotonicFactory(currPrng?: PRNG): ULID {
  if (!currPrng) {
    currPrng = detectPrng();
  }
  let lastTime: number = 0;
  let lastRandom: string;
  return function ulid(seedTime?: number): string {
    if (isNaN(seedTime)) {
      seedTime = Date.now();
    }
    if (seedTime <= lastTime) {
      const incrementedRandom = (lastRandom = incrementBase32(lastRandom));
      return encodeTime(lastTime, TIME_LEN) + incrementedRandom;
    }
    lastTime = seedTime;
    const newRandom = (lastRandom = encodeRandom(RANDOM_LEN, currPrng));
    return encodeTime(seedTime, TIME_LEN) + newRandom;
  };
}
```

関数内で`let lastTime: number = 0`と定義しているので、`monotonicFactory()()`という形で実行してしまうと lastTime が 0 の状態で常に実行されてしまいます。
そうすると、同一ミリ秒の時の生成に関わる以下分岐が、必ず false になってしまいます。
そのため、想定した挙動をしなくなります。
対処法としては、以下のように最初に ULID を生成する関数を取り出しておくことです。

```jsx
import { monotonicFactory } from "ulid";
const generateUlid = monotonicFactory();
for (let i = 0; i < 10; i++) {
  console.log(generateUlid());
}
```

こうすることで、lastTime が 0 始まりにならず、generateUlid 関数を実行する度に上書きされます。
そのため、同一ミリ秒での生成な場合は、Monotonic にできます。
ULID とは直接関係ないですが、内部の処理を分かっておらず上手く動かないと悩んだので、共有しました。

## おわりに

今回は ULID の特に、タイムスタンプとソートについてみていきました。
なんとなくで使っており、上手く行かなかった時にてこずってしまいました。
ただ、今回調べて理解できたので、良かったです。
ULID を、ソートが担保できるユニークな ID として使用する時の参考になれば幸いです。
ここまで読んでいただきありがとうございました。
