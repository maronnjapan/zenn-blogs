---
title: "Typescript周りで便利だなと思ったものの紹介"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Typescript", "Union"]
published: true
---

## はじめに

Typescript 周りを触っていたり記事をみたりして便利だなと思ったけど、それぞれを記事にするほどボリュームをかけなかったものを記載しています。
備忘録的なものですが役立つと思うので、是非見て行ってください。

## オブジェクトのプロパティ一覧をユニオン型の配列にする方法

### コードの全体像

```tsx
const getKeys = <T extends { [key: string]: unknown }>(obj: T): (keyof T)[] => {
  return Object.keys(obj);
};
```

### 簡単な解説

TypeScript のジェネリック型は、柔軟で型安全なコードを書くために重要な機能です。
ジェネリック型は、型をパラメータ化することで、関数やクラスを型に依存せずに定義できます。例えば、以下のような関数を考えてみましょう。

```tsx
function identity<T>(arg: T): T {
  return arg;
}
```

この関数 `identity` は、引数 `arg` の型を `T` としてジェネリック型で定義しています。
関数の戻り値の型も `T` になっています。
これにより、関数を呼び出す際に指定した型の値で動的に型が決定されます。
例えば、以下のように `identity` 関数を呼び出すことができます。

```tsx
const str = identity<string>("Hello");
console.log(str); // Output: 'Hello'
const num = identity<number>(42);
console.log(num); // Output: 42
```

`identity<string>('Hello')` では、`T` が `string` 型に設定され、引数 `'Hello'` がそのまま返されます。
同様に、`identity<number>(42)` では、`T` が `number` 型に設定され、引数 `42` がそのまま返されます。
さらに、型推論を利用することで、型引数を明示的に指定せずに関数を呼び出すこともできます。

```tsx
const str2 = identity("Hello");
console.log(str2); // Output: 'Hello'
const num2 = identity(42);
console.log(num2); // Output: 42
```

この場合、TypeScript が引数の型から型引数 `T` を自動的に推論してくれます。
`str2` は `string` 型、`num2` は `number` 型と推論されます。
また、ジェネリック型には、型の制約（制限）を設定することができます。
これにより、ジェネリック型のパラメータに対して特定の条件を満たすことを要求できます。
以下のように、`extends` キーワードを使って制約を設定します。

```tsx
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

この関数 `getProperty` は、オブジェクト `obj` とそのプロパティ名 `key` を受け取り、対応する値を返します。
ジェネリック型 `T` はオブジェクトの型、`K` はそのオブジェクトのプロパティ名の型を表します。
`K extends keyof T` という制約により、`K` は `T` のプロパティ名のいずれかでなければならないことを保証しています。
試しに先程の getProperty 関数を読んで見ると、第一引数にオブジェクトを指定すると、第二引数はオブジェクトのプロパティを指定するようになります。
このように、ジェネリクス型は型の制約を行うことで、ある程度自由に、しかし一定のルールを持たせることができます
それでは、オブジェクトのプロパティをユニオン型の配列で取得する関数を実装してみましょう。

```tsx
const getKeys = <T extends { [key: string]: unknown }>(obj: T): (keyof T)[] => {
  return Object.keys(obj);
};
```

この関数 `getKeys` は、ジェネリック型 `T` を使ってオブジェクトの型を表します。
`T extends { [key: string]: unknown }` という制約により、`T` はオブジェクト型であることを要求しています。
関数内では、`Object.keys(obj)` を使ってオブジェクトのプロパティ名を文字列の配列として取得しています。
そして、TypeScript の型推論により、`Object.keys(obj)` の戻り値は `(keyof T)[]` 型、つまり `T` のプロパティ名のユニオン型の配列として推論されます。
そのため、明示的なアサーションを使用せずに、目的の型の配列が返されます。
以下のように使用することができます。

```tsx
const person = {
  name: "John",
  age: 30,
  city: "New York",
};
const keys = getKeys(person);
console.log(keys); // Output: ['name', 'age', 'city']
```

`getKeys` 関数に `person` オブジェクトを渡すことで、そのプロパティ名のユニオン型の配列が返されます。

### 参考資料

[https://zenn.dev/ossamoon/articles/694a601ee62526](https://zenn.dev/ossamoon/articles/694a601ee62526)

## ユニオン型の組み合わせを持つ配列を作成する

### 完成したコード

```tsx
type Permutation<T, K = T> = [T] extends [never]
  ? []
  : K extends K
  ? [K, ...Permutation<Exclude<T, K>>]
  : never;
```

[こちら](https://github.com/type-challenges/type-challenges/issues/614#issue-779242337)のコードを丸々コピペしています。

### 簡単な解説

TypeScript の型システムには、ユニオン型、条件型、マップ型、テンプレートリテラル型など、さまざまな高度な型機能が用意されています。その中でも特に強力な機能の一つが、分配条件型（Distributive Conditional Types）です。
分配条件型は、ユニオン型に対して条件型を適用すると、ユニオン型の各構成要素に対して個別に条件型が適用される動作を指します。
つまり、以下のような形式の条件型は分配条件型として扱われます。

```tsx
(A | B | C) extends U ? X : Y
```

この条件型は、以下のように評価されます。

```tsx
(A extends U ? X : Y) | (B extends U ? X : Y) | (C extends U ? X : Y)
```

例えば、以下の条件型を考えてみましょう。

```tsx
type Example<T> = T extends string ? true : false;
```

この条件型に、ユニオン型 `string | number` を適用すると、次のようになります。

```tsx
type Result = Example<string | number>;
```

`Result` の型は、以下のように評価されます。

```tsx
(string extends string ? true : false) | (number extends string ? true : false)
```

`string extends string` は `true` に評価され、`number extends string` は `false` に評価されるので、最終的に `Result` の型は `true | false` になります。
また、TypeScript では再帰的な型の定義も可能です。再帰的な型定義を使用することで、複雑なデータ構造やアルゴリズムを型レベルで表現することができます。
また、TypeScript では再帰的な型の定義も可能です。再帰的な型定義を使用することで、複雑なデータ構造やアルゴリズムを型レベルで表現することができます。
例えば、以下のような再帰的な型定義を考えてみましょう。

```tsx
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};
```

この `DeepReadonly` 型は、オブジェクトのプロパティを再帰的に `readonly` にする型です。
オブジェクトのプロパティがオブジェクトである場合、再帰的に `DeepReadonly` が適用されます。
例えば、以下のような型を考えてみましょう。

```tsx
interface Person {
  name: string;
  age: number;
  address: {
    street: string;
    city: string;
  };
}
```

この `Person` 型に `DeepReadonly` を適用すると、次のようになります。

```tsx
type ReadonlyPerson = DeepReadonly<Person>;
```

`ReadonlyPerson` の型は、以下のように評価されます。

```tsx
{
  readonly name: string;
  readonly age: number;
  readonly address: {
    readonly street: string;
    readonly city: string;
  };
}
```

`address` プロパティもオブジェクトなので、再帰的に `DeepReadonly` が適用され、`address` オブジェクトのプロパティも `readonly` になります。
このように、再帰的な型定義を使用することで、複雑なデータ構造に対して型レベルの操作を行うことができます。
これらの機能を組み合わせることで、`Permutation` 型のような複雑な型定義を行うことができます。

```tsx
type Permutation<T, K = T> = [T] extends [never]
  ? []
  : K extends K
  ? [K, ...Permutation<Exclude<T, K>>]
  : never;
```

`Permutation` 型は、ユニオン型 `T` の全ての順列を表す型を生成します。例えば、`Permutation<'a' | 'b' | 'c'>` は、以下のようなユニオン型になります。

```tsx
["a", "b", "c"] |
  ["a", "c", "b"] |
  ["b", "a", "c"] |
  ["b", "c", "a"] |
  ["c", "a", "b"] |
  ["c", "b", "a"];
```

この型定義では、分配条件型と再帰的な型定義が組み合わされています。
まず、`[T] extends [never] ? [] : ...` の部分で、`T` が空のユニオン型（`never`）の場合に空の配列型（`[]`）を返しています。これがベースケースとなります。
次に、`K extends K ? [K, ...Permutation<Exclude<T, K>>] : never` の部分で、分配条件型が使用されています。ここでは、`K` がユニオン型 `T` の各構成要素に対して個別に適用されます。`K extends K` は常に `true` になりますが、分配条件型の動作により、`K` がユニオン型の各構成要素に対して個別に適用されるのです。
例えば、`Permutation<'a' | 'b' | 'c'>` の場合、最初のステップでは `K` は `'a' | 'b' | 'c'` になります。そして、分配条件型の動作により、以下のように評価されます。

```tsx
('a' extends 'a' | 'b' | 'c' ? ['a', ...Permutation<Exclude<'a' | 'b' | 'c', 'a'>>] : never) |
('b' extends 'a' | 'b' | 'c' ? ['b', ...Permutation<Exclude<'a' | 'b' | 'c', 'b'>>] : never) |
('c' extends 'a' | 'b' | 'c' ? ['c', ...Permutation<Exclude<'a' | 'b' | 'c', 'c'>>] : never)
```

これにより、`'a'`、`'b'`、`'c'` のそれぞれに対して、再帰的に `Permutation` 型が適用されます。
`[K, ...Permutation<Exclude<T, K>>]` の部分では、現在の構成要素 `K` を配列の先頭に置き、残りの要素に対して再帰的に `Permutation` 型を適用しています。`Exclude<T, K>` は、`T` から `K` を除外した残りの要素を表すユニオン型です。
この再帰的な処理により、ユニオン型 `T` のすべての順列が生成されます。
以下は、`Permutation<'a' | 'b' | 'c'>` の評価の流れを示しています。

1. `'a'` の場合:
   - `['a', ...Permutation<Exclude<'a' | 'b' | 'c', 'a'>>]`
   - `['a', ...Permutation<'b' | 'c'>]`
   - `['a', 'b', 'c'] | ['a', 'c', 'b']`
2. `'b'` の場合:

   - `['b', ...Permutation<Exclude<'a' | 'b' | 'c', 'b'>>]`
   - `['b', ...Permutation<'a' | 'c'>]`
   - `['b', 'a', 'c'] | ['b', 'c', 'a']`

3. `'c'` の場合:
   - `['c', ...Permutation<Exclude<'a' | 'b' | 'c', 'c'>>]`
   - `['c', ...Permutation<'a' | 'b'>]`
   - `['c', 'a', 'b'] | ['c', 'b', 'a']`

最終的に、これらのユニオン型が結合されて、`['a', 'b', 'c'] | ['a', 'c', 'b'] | ['b', 'a', 'c'] | ['b', 'c', 'a'] | ['c', 'a', 'b'] | ['c', 'b', 'a']` となります。
このように、`Permutation` 型は分配条件型と再帰的な型定義を組み合わせることで、ユニオン型の全ての順列を表現しています。分配条件型は、ユニオン型の各構成要素に対して条件型を個別に適用する強力な機能であり、再帰的な型定義と組み合わせることで、複雑な型レベルのアルゴリズムを実現することができます。

### 参考資料

[https://mosya.dev/blog/tc-permutation](https://mosya.dev/blog/tc-permutation)
[https://github.com/type-challenges/type-challenges/issues/614](https://github.com/type-challenges/type-challenges/issues/614)

## オブジェクトの配列で任意のプロパティをキーにして、一致するオブジェクトを結合してオブジェクトの配列を作りなおす

### 完成したコード

```tsx
type ObjectType = {
  [key: string]: string;
} & { id: string };
const a: ObjectType[] = [
  { id: "id1", test: "test1" },
  { id: "id1", test: "test2", test2: "test4", test3: "test3" },
  { id: "id2", test: "test3" },
  { id: "id1", test2: "test4" },
  { id: "id1", test2: "test5" },
  { id: "id1", test2: "test6" },
  { id: "id1", test4: "test7" },
  { id: "id1", test5: "test8" },
];
type ResultType = { [key: string]: string | string[] };
const b = a.reduce((acc, curr) => {
  const existingObj = acc.find((obj) => obj.id === curr.id);
  if (!existingObj) {
    return [...acc, { ...curr }];
  }
  Object.entries(curr).forEach(([key, value]) => {
    if (key !== "id") {
      const preVal = existingObj[key];
      if (preVal === undefined) {
        existingObj[key] = value;
      } else {
        existingObj[key] = Array.isArray(preVal)
          ? [...preVal, value]
          : [preVal, value];
      }
    }
  });
  return acc;
}, [] as ResultType[]);
// Output
[
  {
    id: "id1",
    test: ["test1", "test2"],
    test2: ["test4", "test4", "test5", "test6"],
    test3: "test3",
    test4: "test7",
    test5: "test8",
  },
  {
    id: "id2",
    test: "test3",
  },
];
```

### 軽い解説

TypeScript を使用すると、オブジェクトの配列を柔軟に操作することができます。
ここでは、オブジェクトの配列から任意のプロパティをキーにして、同じキーを持つオブジェクトを結合し、新しいオブジェクトを生成する方法について解説します。
まず、オブジェクトの型を定義します。ここでは、`id`プロパティを必須とし、その他のプロパティは文字列型とします。

```tsx
type ObjectType = {
  [key: string]: string;
} & { id: string };
```

次に、サンプルデータとして、`ObjectType`型の配列を用意します。

```tsx
const a: ObjectType[] = [
  { id: "id1", test: "test1" },
  { id: "id1", test: "test2", test2: "test4", test3: "test3" },
  { id: "id2", test: "test3" },
  { id: "id1", test2: "test4" },
  { id: "id1", test2: "test5" },
  { id: "id1", test2: "test6" },
  { id: "id1", test4: "test7" },
  { id: "id1", test5: "test8" },
];
```

結合後のオブジェクトの型を定義します。プロパティの値は、文字列型または文字列型の配列とします。

```tsx
type ResultType = { [key: string]: string | string[] };
```

それでは、`reduce`メソッドを使用して、配列を結合していきます。

```tsx
const b = a.reduce((acc, curr) => {
  const existingObj = acc.find((obj) => obj.id === curr.id);
  if (!existingObj) {
    return [...acc, { ...curr }];
  }
  Object.entries(curr).forEach(([key, value]) => {
    if (key !== "id") {
      const preVal = existingObj[key];
      if (preVal === undefined) {
        existingObj[key] = value;
      } else {
        existingObj[key] = Array.isArray(preVal)
          ? [...preVal, value]
          : [preVal, value];
      }
    }
  });
  return acc;
}, [] as ResultType[]);
```

`reduce`メソッドのコールバック関数では、まず`find`メソッドを使用して、現在処理しているオブジェクト（`curr`）と同じ`id`を持つオブジェクトが、結果の配列（`acc`）内に存在するかどうかを確認します。
存在しない場合は、現在のオブジェクトを新しいオブジェクトとして結果の配列に追加します。
存在する場合は、`Object.entries`メソッドを使用して、現在のオブジェクトのプロパティを反復処理します。
`id`プロパティは今回識別子となるので、プロパティは必ず存在していますし、配列にもしたくないので除外します。
その他のプロパティについては既存のオブジェクトに存在するかどうかを確認します。
存在しない場合は、プロパティと値を直接オブジェクトに追加します。
存在する場合は、プロパティの値が配列であるかどうかを確認します。配列の場合は、スプレッド構文を使用して新しい値を追加します。配列でない場合は、新しい配列を作成します。
最終的に、`reduce`メソッドは結合後のオブジェクトの配列を返します。
このようにして、TypeScript を使用してオブジェクトの配列を任意のプロパティをキーにして結合することができます。
同じキーを持つオブジェクトが複数ある場合は、プロパティの値を配列として扱い、単一の値の場合は文字列として扱います。

## おわりに

今回は Typescrpt 周りで役立ちそうなネタを記載しました。
中々便利なものが世の中には存在するなと感じています。
ここまで読んでいただきありがとうございました。
