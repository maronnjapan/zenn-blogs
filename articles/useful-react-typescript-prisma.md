---
title: "React,Typescript,Prismaで役立ったもの、気になったもの"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React", "Typescript", "Prisma"]
published: true
---

## はじめに

Typescript や React,Prisma の記事など見て、実際に便利だなと思ったものを記載しています。
すでに知っている部分もあるかもしれませんが、何かの参考になれば幸いです。

## サクッと React で特定のコンポーネントを再レンダリングさせる

ボタンをクリックしたら、特定のコンポーネントを再描画させたいというケースはあると思います。
具体的には、更新ボタンをクリックしたとき内容を最新化させたいが、画面全体のリロードを避けたいといったケースです。
そんな時調べれば色々出てくるのですが、今回は限りなくサクッとできる方法を紹介します。
具体的には以下のように key 属性を使用した state 管理です。

```tsx
import { useState } from "react";
export default function Page() {
  const [key, setKey] = useState(0);
  return (
    <>
      <button onClick={() => setKey((prev) => prev + 1)}>更新</button>
      <div key={key}>レンダリング</div>
    </>
  );
}
```

key 属性はリストのループ処理に、コンポーネントを一意に定めるためよく使用されます。
この key 属性を使用すれば、リストが変更されたときに、変更された箇所だけをコンポーネントの再構築を行います。
であれば、ループ内で無くても key 属性を指定し、それを変更すればレンダリングできそうです。
実際にやってみたら出来ました。
レンダリングさせるコンポーネントと更新ボタンを配置したコードはそれぞれ以下の通りです。
レンダリングさせるコンポーネント

```tsx
"use client";
import { useState } from "react";
export default function TestContent() {
  const [text, setText] = useState("ローディング中");
  setTimeout(() => {
    setText(() => "こんにちは");
  }, 1000);
  return <div>{text}</div>;
}
```

呼び出しコンポーネント

```tsx
"use client";
import { useState } from "react";
import TestContent from "./_components/TestContent";
export default function Page() {
  const [key, setKey] = useState(0);
  return (
    <div style={{ padding: "2rem" }}>
      <button
        onClick={() => setKey((prev) => prev + 1)}
        style={{ border: "2px solid" }}
      >
        更新
      </button>
      <TestContent key={key}></TestContent>
    </div>
  );
}
```

これを画面で確認すると以下のようになります。
![btn-click.gif](/images/useful-react-typescript-prisma/btn-click.gif)
ちゃんとレンダリングされていそうですね。
是非とも試してみてください。
**参考資料**
[https://qiita.com/putan/items/8d976afab638ffb96acb](https://qiita.com/putan/items/8d976afab638ffb96acb)

## Typescript5.5 での気になる機能

Zenn とか見てて 6 月にリリースされる Typescript 5.5 で気になった機能を記載します。

### filter 関数の自動型推論

早速ですが以下のコードをみてください。

```tsx
const list = ["test1", undefined, "test2"];
const filterList = list.filter((l) => l !== undefined);
```

このとき変数 filterList の型はどうなるでしょうか？
自然に考えると`string[]`ですね。
しかし、実際は添付画像のように`(string | undefined)[]`となります。
![Untitled](/images/useful-react-typescript-prisma/Untitled.png)
いや、undefine 明示的に取り除いているし、undefined 型が残るのはおかしいですやんとずっと思っていました。
ですが、現実 undefined は残っているので、これまでは以下のように対応していました。

```tsx
const list = ["test1", undefined, "test2"];
const filterList = list.filter((l) => l !== undefined) as string[];
//or
const list = ["test1", undefined, "test2"];
const filterList = list.filter((l): l is string => l !== undefined);
```

アサーションや型ガードを自分で記載するしかなく、いつか間違えそうという不安を抱えていました。
ところが、この度 5.5 では以下の Pull Request がマージされていることから分かるように、この問題が解消されます!!!!!!!!!!!!!!!!!!
[https://github.com/microsoft/TypeScript/pull/57465](https://github.com/microsoft/TypeScript/pull/57465)
早速試してみましょう。
以下 Typescript の Playground をみてください。
[https://www.typescriptlang.org/play?ts=5.5.0-dev.20240424#code/MYewdgzgLgBANgS2jAvDA2gcigU2gRkwBoYBXMAExwDMEwcKTs8oAmTAXQChRJZa4uAE4AZJLDSJoAOgHCAFHFQA+eDACEKNOSq16FAJQBuIA](https://www.typescriptlang.org/play?ts=5.5.0-dev.20240424#code/MYewdgzgLgBANgS2jAvDA2gcigU2gRkwBoYBXMAExwDMEwcKTs8oAmTAXQChRJZa4uAE4AZJLDSJoAOgHCAFHFQA+eDACEKNOSq16FAJQBuIA)
変数 filterList へホバーすると、なんと型ガードやアサーションをしていないのに undefined が除去されています。
![2024-04-25_00h27_20.png](/images/useful-react-typescript-prisma/2024-04-25_00h27_20.png)
これは感動ものですね。
今後は filter 関数による値除去がよりはかどりそうです。
**参考資料**
[https://zenn.dev/ubie_dev/articles/ts-infer-type-predicates](https://zenn.dev/ubie_dev/articles/ts-infer-type-predicates)
[https://github.com/microsoft/TypeScript/pull/57465/files](https://github.com/microsoft/TypeScript/pull/57465/files)

### オブジェクトのプロパティについて順番の整合性を担保できるようになった

typescript は以下のように型定義を行い、プロパティを制限することができます。

```tsx
interface Test {
  x: number;
  y: number;
  z: number;
}
```

これはもちろん便利で、似たような型定義は日々使用しています。
しかし、この型定義では x,y,z プロパティがどの順番で定義しても問題ありません。
基本的にそれでも困らないのですが、例えば以下のように値の大小を比べる処理があるとします。

```tsx
const test = { x: 2, y: 1 };
console.log(Object.values(test).reduce((acc, cur) => acc > cur));
//true
const test = { y: 1, x: 2 };
console.log(Object.values(test).reduce((acc, cur) => acc > cur));
//false
```

このようにオブジェクトの定義する準備で異なる値が出力されてしまいます。
そもそもこのコードの是非は置いておいて、こういったオブジェクトの順番によって値が変わる処理はあります。
その問題を解消する機能が v5.5.5 でリリースされる?予定です。
それが in that order です。
以下のように型定義の後ろに付与すると、プロパティの定義順を制限できます。

```tsx
interface Test {
	x:number;
	y:number;
	z:number;
} in that order
```

この定義を行うと

```tsx
const a: Test = { x: 1, y: 2, z: 3 };
```

は定義ですますが、

```tsx
const b: Test = { y: 1, x: 2, z: 3 };
```

は定義している順番が異なるので、エラーが発生します。
中々面白い機能です。
具体的な活用法はまだぼんやりしていますが、オブジェクトのプロパティの順番が担保されるなら、片方のオブジェクトについて`Object.keys()`したものをもう片方のオブジェクトに使用できるなど、それなりの用途はありそうです。
対応予定が v5.5.5 らしいので、まだ Playground では使用できませんが、確認出来次第試していおくと思います。
**参考資料**
[https://qiita.com/uhyo/items/787a475bb618811d3771](https://qiita.com/uhyo/items/787a475bb618811d3771)
[https://github.com/microsoft/TypeScript/issues/58019](https://github.com/microsoft/TypeScript/issues/58019)

## 型定義から Zod のスキーマを作る時いい感じに補完など効かせてほしい

※[こちらの記事](https://zenn.dev/uttk/articles/bd264fa884e026#%E5%9E%8B%E5%BC%95%E6%95%B0%E3%81%AE%E6%B8%A1%E3%81%97%E6%96%B9)のコードを丸々拝借しています。
値のバリエーションを行うライブラリは多く存在します。
その一つに[Zod](https://zod.dev/)があります。
Github のスター数から見るにかなり使用されていることがわかりますね。
この Zod を使って、バリエーションをする際 Typescript の型定義を満たす値かを判定したいことは往々にしてあります。
具体的には以下のようなコードです。

```tsx
import * as z from "zod";
type Test = { test: string };
const validate = z.object({
  test: z.string(),
});
const val: Test = { test: "test" };
const result = validate.safeParse(val);
```

こうすれば、値がちゃんと指定の条件を満たした値かを判定できます。
ですが、このやり方は Zod のスキーマを作る時に補完が効かない問題があります。
例えば validate の部分を以下のように変えたとしても、エラーが発生せずコードが実行されるまで分かりません。

```tsx
const validate = z.object({
  test2: z.string(),
});
```

また、test プロパティの型は string なので、Zod でスキーマを作るときも string に限定したいです。
ですが、こちらも以下のように number を指定しても実行するまで分かりません。

```tsx
const validate = z.object({
  test: z.number(),
});
```

少し使いにくいですね。
なので、z.object を使用した時にプロパティに補完を効かせつつ、型定義に対応する Zod スキーマしか許容しないように修正します。
具体的には以下の通りです。

```tsx
import * as z from "zod";
type toZod<T extends Record<string, any>> = {
  [K in keyof T]-?: z.ZodType<T[K]>;
};
type Test = { test: string };
const validate = z.object<toZod<Test>>({
  test: z.string(),
});
const val: Test = { test: "test" };
const result = validate.safeParse(val);
```

注目するのは

```tsx
type toZod<T extends Record<string, any>> = {
  [K in keyof T]-?: z.ZodType<T[K]>;
};
```

です。
toZod については以下のことを行っています。
① オブジェクト型のプロパティを toZod 型のプロパティに限定します。
② 値の型定義はオブジェクト型の値の型を定義し、それを Zod のスキーマのタイプにしています。
もう少し中を見てみます。
ジェネリック型`T`は、`Record<string, any>`によって制約されており、これはキーが文字列型で値が任意の型を持つオブジェクト型を表します。
つまり、`T`はこの制約を満たすオブジェクト型であることが求められます。
`[K in keyof T]`は、mapped types と呼ばれる構文で、`keyof T`は`T`のすべてのプロパティ名（キー）のユニオン型を表し、`K in keyof T`は`T`のプロパティ名を一つずつ`K`に割り当てながらループするような処理を表現しています。
これにより、`T`のすべてのプロパティを新しい型定義のプロパティとして使用することができます。
`-?`は、オプショナルプロパティを許容しないことを表します。
通常、`?`を付けるとプロパティをオプショナルにできますが、`?`とすることでオプショナルプロパティを許容せず、必ず定義しなければならないことを強制できます。
`T[K]`は、`T`のプロパティ`K`に対応する値の型を表し、元のオブジェクト型`T`の各プロパティの値の型を取得することができます。
`z.ZodType<T[K]>`は、Zod ライブラリの型定義を用いて、`T[K]`の型に対応する Zod スキーマの型を指定しています。
`z.ZodType`は、Zod の各スキーマ型（例えば、`z.string`、`z.number`など）に対応する型を表現するユーティリティ型です。
まとめると、この型定義`toZod<T>`は、オブジェクト型`T`を受け取り、そのオブジェクトの各プロパティに対応する Zod スキーマの型を持つ新しいオブジェクト型を返します。
また、`-?`を使うことで、すべてのプロパティが必須であることを保証しています。
これにより、TypeScript の型定義から Zod スキーマへの変換を型安全に行うことができます。
ただ、一点結構致命的な問題はあります。
それは、ネストしたオブジェクトに対しては使用できない点です。
なので、実際に使用する場合はネストしていないオブジェクトに対してのみ使うようにしてください。
**参考資料**
[https://zenn.dev/uttk/articles/bd264fa884e026#型引数の渡し方](https://zenn.dev/uttk/articles/bd264fa884e026#%E5%9E%8B%E5%BC%95%E6%95%B0%E3%81%AE%E6%B8%A1%E3%81%97%E6%96%B9)
[https://zenn.dev/ynakamura/articles/65d58863563fbc#discuss](https://zenn.dev/ynakamura/articles/65d58863563fbc#discuss)

## Prisma のテーブル GetPayload を使ってネストしたテーブルの型定義を簡単に行う方法

Prisma は、Node.js と TypeScript のためのオープンソースの ORM であり、データベースとのやり取りを簡単に行うことができます。
Prisma はそれなりに便利ですが、戻り値などを定義する時取得するテーブルの型が深くまでネストしていた場合、その値を
Prisma には、`テーブル名GetPayload`という便利な型ユーティリティが用意されており、これを使用することでネストしたテーブルの型定義を簡単に行うことができます。

### テーブル名 GetPayload とは？

`テーブル名GetPayload`は、Prisma クエリの戻り値の型を取得するためのユーティリティ型です。
これを使用することで、Prisma クエリの結果の型を型安全に取得することができます。
具体的な使用方法を見てみましょう。

### ネストしたテーブルの型定義

先程の`テーブル名GetPayload`による型定の具体例を見てみましょう。
例えば、以下のような Prisma スキーマがあるとします。

```
model User {
  id    Int     @id @default(autoincrement())
  name  String
  posts Post[]
}
model Post {
  id      Int    @id @default(autoincrement())
  title   String
  content String
  author  User   @relation(fields: [authorId], references: [id])
}
```

この場合、`User`テーブルと`Post`テーブルの間には 1 対多のリレーションシップがあります。
このようなネストしたテーブルの型定義を行う場合、`テーブル名GetPayload`を使用すると簡単に行うことができます。

```tsx
type UserWithPosts = Prisma.UserGetPayload<{
  include: { posts: true };
}>;
```

ここでは、`UserWithPosts`という型を定義しています。
この型は、`Prisma.UserGetPayload`を使用して、`User`テーブルの情報に加えて、関連する`Post`テーブルの情報も含むように設定しています。
これを使用することで、include したテーブルの定義を以下のように自分で書く必要がなくなります。

```tsx
type UserData = User & { posts: Post[] };
```

結構便利ですね。
さらに、`テーブル名GetPayload`を使用することで、より深くネストしたテーブルの取得処理でも型定義を簡単に行うことができます。
例えば、以下のような Prisma クエリがあるとします。

```tsx
typescriptCopy codeconst userWithPosts = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: {
      include: {
        comments: true,
      },
    },
  },
});
```

この場合、`userWithPosts`の型は以下のように定義することができます。

```tsx
type UserWithPostsAndComments = Prisma.UserGetPayload<{
  include: {
    posts: {
      include: {
        comments: true;
      };
    };
  };
}>;
```

ここでは、`UserWithPostsAndComments`という型を定義しています。この型は、`Prisma.UserGetPayload`を使用して、`User`テーブルの情報に加えて、関連する`Post`テーブルと`Comment`テーブルの情報も含むように設定しています。
このように、`テーブル名GetPayload`を使用することで、ネストしたテーブルの取得処理でも型定義を簡単に行うことができます。これにより、型安全性を確保しつつ、複雑なデータ構造を扱うことができます。
Prisma から受け取った値を変換する Mapper 関数の引数を型定義するときとか、テストコードで比較する値を定義する時に便利なので、是非ご活用下さい。
**参考資料**
[https://github.com/prisma/prisma/discussions/10928#discussioncomment-1920961](https://github.com/prisma/prisma/discussions/10928#discussioncomment-1920961)

## おわりに

今回は気になった機能などについての話でした。
React と Prisma に関しては開発でもそれなりに活用しており、非常に便利だなと感じています。
今後もこういった便利機能を収集して、共有できたらと思います。
ここまで読んでいただきありがとうございました。
