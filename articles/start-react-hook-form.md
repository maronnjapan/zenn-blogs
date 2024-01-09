---
title: "React Hook Formの第一歩"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React", "typescript"]
published: true
---

## はじめに

React には入力フォームの検証を行うことができるライブラリが存在します。
このライブラリを使うことで、何度も同じような検証ロジックを書く必要がなくなります。
今回は数あるライブラリの中で、[React Hook Form](https://react-hook-form.com/)について見ていきます。
React Hook Form を選んだのは、フォーム検証モジュールで最初に思い浮かぶものを考えた時に、頭の中に出てきたからです。
あまり複雑なことはやらず、React Hook Form をとりあえず使えることを重視しています。
では始めます。

## React Hook Form のクイックスタート

まずはインストールです。
`npm install react-hook-form`でモジュールをインストールします。
次に Ract Hook Form を使って最低限動くコンポーネントを以下に示します。

```tsx
import { useForm } from "react-hook-form";
export default function ReactHookForm() {
  const defaultValues = {
    name: "名無し",
    email: "test@example.com",
    gender: "male",
    memo: "",
  };
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm({ defaultValues });
  const nameRequired = {
    ...register("name", {
      required: "名前は必須です",
    }),
  };
  const onSubmit = (data: typeof defaultValues) => console.log(data);
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label htmlFor="name">名前:</label>
        <input id="name" type="text" {...nameRequired}></input>
        <div>{errors.name?.message}</div>
      </div>
      <div>
        <button type="submit">送信</button>
      </div>
    </form>
  );
}
```

最低限動かすために必要なのは以下の通りです。

- React Hook Form で検証する値の初期値を設定する
- React Hook Form の機能を使うために、初期値を設定しながら useForm を呼び出す。
- register を使用して、行うバリデーションを設定する。
- 検証が成功した時の処理を定義する。
- form 要素に onSubmit 属性を追加し、handleSubmit を設定する。設定する際に、検証が成功した時の処理を引数に渡す。
- formState 内の errors オブジェクトを取得して、エラーが発生した時のメッセージを出せるようにする。

こうすれば、input 要素が空の状態で送信しようとしても以下画像のようにエラーが発生します。
![2023-12-23_13h23_39.png](/images/start-react-hook-form/2023-12-23_13h23_39.png)
やっていることは、useForm に初期値を設定して、検証ルールを作成することだけです。
なので、使うのはとても簡単そうです。
ただ、折角なのでもう少し内部の値について見ていきます。
まずは useForm そのもののオプションについてです。

## useForm のオプション達

### **defaultValues**

useForm で検証する値の初期値です。
基本的にここの理解で詰まることはなさそうですが、[こちらの記事](https://zenn.dev/yodaka/articles/e490a79bccd5e2)であるように API から値を設定する場合は値が確定するまでは待つ処理が必要となりそうです。

### **mode**

検証を実行するタイミングを設定するオプションです。
値としては以下のものがあります。

- onChange: 入力要素が変更される都度検証(パフォーマンスを悪化させる恐れあり)
- onBlur: 入力要素からフォーカスが外れたとき
- onSubmit: フォームが送信されたとき。ただし、一度送信してエラーが発生した以降は onChange モードで検証するようになります。
- onTouched: 最初の onBlur のタイミングで検証します。
- all onBlur/onChange 両方の時に検証します。

なお、設定した mode ですが、それぞれのフォームで実行されたイベントを共有ることはありません。
具体的には以下のように二つの input 要素があって、mode が onTouched の時を考えます。

```tsx
<input id="name" type="text" {...nameRequired}></input>
<input id="email" type="text" {...emailRequired}></input>
```

最初の input 要素で検証された後、イベントを共有するなら二つ目の input 要素を入力している時すでに検証が始まります。
しかし、React Hook Form はイベントを共有しないので、実際は二つ目の input 要素も onBlur のタイミングで検証が開始され、それ以降 onChange で検証するようになります。

### **reValidateMode**

送信の時エラーが発生した場合、再検証するタイミングを設定するオプションです。
設定値としては、onChange,onBlur,onSubmit があり、デフォルトは onChange です。

### **criteriaMode**

エラーの保持方法について設定するオプションです。
最初のエラーだけを集める「firstError」と全てのエラーを集める「all」があります。
デフォルト値は「firstError」です。
このオプションはなんとなくやっていることは分かりますが、実際どう活用するのかが分かりにくいのでもう少し解説します。
両者の違いですが、formState 内の errors オブジェクトに types プロパティを付与するかどうかということになります。
それを示すために以下のコードを見ていきます。

```tsx
import { useForm } from "react-hook-form";
export default function ReactHookForm() {
  const defaultValues = {
    name: "名無し",
    email: "test@example.com",
    gender: "male",
    memo: "",
  };
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm({ defaultValues, criteriaMode: "all" });
  const emailRequired = {
    ...register("email", {
      maxLength: { value: 3, message: "文字数オーバーです" },
      pattern: {
        value: /[0-9]+/i,
        message: "形式が違います",
      },
    }),
  };
  const onSubmit = (data: typeof defaultValues) => console.log(data);
  return (
    <form onSubmit={handleSubmit(onSubmit, (errors) => console.log(errors))}>
      <div>
        <label htmlFor="email">メールアドレス:</label>
        <input id="email" type="text" {...emailRequired}></input>
        <div>{errors.email?.message}</div>
        <div>{errors.email?.types?.maxLength && "文字数が多いですね"}</div>
        <div>
          {errors.email?.types?.pattern && "形式が所定のものじゃないです。"}
        </div>
      </div>
      <div>
        <button type="submit">送信</button>
      </div>
    </form>
  );
}
```

変数 emailRequired でバリデーションを複数設定しているコンポーネントとなっています。
そして、今回 criteriaMode の値は「all」にしています。
また、エラーメッセージは message プロパティの値と、types プロパティに指定のバリエーションタイプがある場合表示するようにしています。
これで例えばフォームに「aaaaa」という値を入力して送信ボタンをクリックしてください。
![2023-12-23_14h21_16.png](/images/start-react-hook-form/2023-12-23_14h21_16.png)
すると画像のようにエラーメッセージが複数表示され、console.log にも types プロパティが付与している状態で表示されています。
次に criteriaMode の値を「firstError」に変更して、再度送信してみます。
![2023-12-23_14h24_08.png](/images/start-react-hook-form/2023-12-23_14h24_08.png)
今度はエラーメッセージが message プロパティのものしか表示されず、console.log にも types プロパティが表示されていません。
このように criteriaMode の値の違いはエラー対象となった type の集まりである types プロパティを設定するか、しないかを決めることです。
なので、criteriaMode を使用する際の注意点として、errors オブジェクトの message プロパティにエラーが複数格納されるわけではないということです。
message プロパティはあくまで一番最初に検証がエラーになった設定のメッセージしか格納しません。
そのため、複数のエラーメッセージを表示させるには criteriaMode の値を「all」にしつつ、`{errors.email?.types?.maxLength && '文字数が多いですね'}`といった別途画面に表示させるための条件分岐は必要となります。

### **houldUseNativeValidation**

[クライアント側のフォーム検証](https://developer.mozilla.org/ja/docs/Learn/Forms/Form_validation)を活用しつつ、React Hook Form のバリデーションを設定するかを決めるオプションです。
デフォルト値は false になっています。
これもイメージがわきにくいので、例を示します。
以下のコードをみてください。

```tsx
import { useForm } from "react-hook-form";
export default function ReactHookForm2() {
  const { register, handleSubmit } = useForm({
    shouldUseNativeValidation: true,
  });
  const nameRequired = {
    ...register("name", {
      required: "名前は必須です",
    }),
  };
  const onSubmit = (data: any) => console.log(data);
  return (
    <form onSubmit={handleSubmit(onSubmit, (errors) => console.log(errors))}>
      <div>
        <label htmlFor="name">名前:</label>
        <input {...nameRequired}></input>
      </div>
      <div>
        <button type="submit">送信</button>
      </div>
    </form>
  );
}
```

register メソッドを使って必須チェックを行っていますね。
さらに shouldUseNativeValidation の値を true にしています。
これを画面に表示され、フォームの値を空で送信すると、以下画像のように馴染みあるエラーメッセージが表示されます。
![Untitled](/images/start-react-hook-form/Untitled.png)
次に shouldUseNativeValidation の値を false にして実行してみると、バリデーションエラーで送信はできませんが、先程でた必須である旨の要素が表示されません。
このようにブラウザ側で用意されている検証機能と一致する検証設定を React Hook Form で行った場合、その機能を使用するかを決めるのが shouldUseNativeValidation オプションです。
正直、React Hook Form を使っているならブラウザ側で用意した検証機能を使う意味はそこまでないので、特に設定する必要はないかなと思います。

### **shoulFocusError**

エラー発生した時にエラー対象のフォームにフォーカスを移動させるかを決めるオプションです。
デフォルト値は true です。

### **delayError**

エラーを表示させるための遅延時間を設定します。

### **resolver**

外部検証ライブラリを適切するオプションです。
こちらは後ほど実際に使用するので、一旦説明は以上とします。

## useForm から受け取る値

ここでは useForm から受け取る各値について見ていきます。

### **register**

フォーム要素に紐づけるイベントハンドラーなどを含めたオブジェクトです。
基本的に検証する内容はここで決定します。
構文は以下の通りです。

```tsx
register(
  "チェックしたいフォームのフィールド名",
  検証や値の設定などのオプション
);
```

フィールド名は typescript を使っており、defaultValue を定義しているなら、補完で出てくるのでそれを選びます。
オプションについては以下のものがあります。

- 初期設定
  - value 入力値。ただ、defaultValue を定義すればそれば初期値となるので、設定しなくてもよい
  - disabled フォームの入力を無効にする。(デフォルトは false)
  - onChange change イベントのハンドラー
  - onBlur blur イベントのハンドラ
- 検証設定
  - required 必須チェック
  - maxLength 最大文字列長
  - minLength 最小文字列長
  - max 最大値
  - min 最小値
  - pattern 入力値を正規表現でチェック
  - validate 独自検証を追加する
- 送信時の値変換設定
  - valueAsNumber 入力値を Number に変換して渡す
  - valueAsDate 入力値を日付型にして渡す
  - setValueAs 入力値を加工して渡す関数

検証設定においては、validate と required 以外は

```tsx
{
	value:'制限する値',
	message:'エラー時表示されるメッセージ'
}
```

で具体的にチェックする内容を設定します。
required は表示させるエラーメッセージだけ設定すれば問題ありません。
validate については、以下のようにして独自の検証を行うことができます。

```tsx
const nameRequired = {
  ...register("name", {
    validate: {
      check: (val, formValues) => {
        return !val.includes("ダメ") || "不適切な言葉が含まれています。";
      },
    },
  }),
};
```

なお、関数の第一引数は対象のフォームの値で、第二引数は対象のフォームを含めた全ての defaultValue で設定したフォームの値が格納されています。
register で検証などの内容を定義したら、HTML 要素へはスプレッド構文で渡せば設定内容が反映されます。
それは register 関数の戻り値が ref,name,onChange,onBlur などの属性を含むためです。

### formState

フォームの状態を示すオブジェクトです。
取得できるものは以下の通りです。

- errors 検証が失敗した時のエラー情報
- isDirty ユーザーがいずれかのフォーム要素を変更したかを boolean で返す
- dirtyFields ユーザーが変更したフィールド情報を「フィールド名:値」で返す。
  - 注意点として、一回の検証で操作した時に値を返すのではなく、その画面で一回でもフォームの操作を行ったら、以降はずっと操作したフィールドとして認識されます。
- touchedFields ユーザーが操作したフィールドを「フィールド名:true」で返す。
  - 注意点として、一回の検証で操作した時に値を返すのではなく、その画面で一回でもフォームの操作を行ったら、以降はずっと操作したフィールドとして認識されます。
- defaultValues 設定した初期値
- isSubmitted フォームが送信されたかを boolean で返す
- isSubmitSuccessful フォームが正常に送信されたかを boolean で返す
- isSubmitting フォームが送信中かを boolean で返す
- isLoading フォームがロード中か(非同期のデフォルト値を読み込んでいるか)を boolean で返す
- submitCount フォームが送信された回数
- isValid フォームの入力値正しいか
- isValidating フォームが検証中であるか

一覧を紹介したので、その中の一部を使って活用例を見ていきます。
errors オブジェクトはエラーメッセージを出すのに先程使用したので、割愛します。
活用例はボタンの非活性についてです。
以下のように検証が問題あるときや、送信中の場合はボタンを押せないようにするなどできます。

```tsx
<button type="submit" disabled={!isValid || isSubmitting}>
  送信
</button>
```

これによって、連打対策などができます。

### watch

指定された要素の変更を監視する関数です。
監視する要素を指定しない場合は React Hook Form の範囲内の全てのフォーム要素を監視します。
感覚としては State のように扱うことができます。

### **getValues**

指定したフォーム要素の値を取得する関数です。
値を指定しなければ、全てのフォーム要素の値を返します。
こちらも State ぽく扱えるので、正直 watch との違いがあまり分かっていないです。

### **handleSubmit**

サブミットの時に呼び出す関数です。
第一引数に送信が成功した時の関数を定義し、第二引数に送信が失敗した時の関数を定義します。
送信が失敗した時の関係については設定が任意となっています。

## 検証ライブラリと連携する

最後に検証用のライブラリを React Hook Form に組み込むことで、検証設定をライブラリに任せることを行います。
検証ライブラリはライブラリとして提供されているだけあって、複雑な操作もできる機能が豊富にあります。
さらに、検証ライブラリを React Hook Form に取り入れることはとても簡単なので、よっぽど単純なことしかやらないとか特別な事情がない限りは、導入するのが個人的には良いと思います。
では早速始めます。
まず検証ライブラリとして[Yup](https://github.com/jquense/yup)をインストールします。
Yup 以外には[Zod](https://zod.dev/)というライブラリもありますし他にも様々なライブラリが提供されていますが、今回はとりあえず Yup を使います。
Yup と Zod の違いなどについては[こちらの記事](https://zenn.dev/wintyo/articles/6122304cb56c86)など詳細にまとめられたものがあるので、参照ください。
`npm i @hookform/resolvers yup`を実行します。
インストールができたら、以下のようなコンポーネントを実装します。

```tsx
import { yupResolver } from "@hookform/resolvers/yup";
import { useForm } from "react-hook-form";
import * as yup from "yup";
const schema = yup.object({
  name: yup
    .string()
    .required(`名前は必須入力です`)
    .max(20, `名前は20文字以内で入力してください`),
});
export default function ReactHookForm() {
  const defaultValues = {
    name: "名無し",
  };
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm({ defaultValues, resolver: yupResolver(schema) });
  const onSubmit = (data: typeof defaultValues) => console.log(data);
  return (
    <form onSubmit={handleSubmit(onSubmit, (errors) => console.log(errors))}>
      <div>
        <label htmlFor="name">名前:</label>
        <input {...register("name")}></input>
        <div>{errors.name?.message}</div>
      </div>
      <div>
        <button type="submit">送信</button>
      </div>
    </form>
  );
}
```

schema を定義して、それを resolver オプションに yupResolver 関数を使いつつ渡せば後はこれまでと同様にバリデーションチェックを行うことができます。
とても簡単ですね。
バリデーションの内容については Yup が提供したものを使うのもよいですし、以下のような定義をして独自のバリデーションを設定することもできます。

```tsx
const schema = yup.object({
  name: yup
    .string()
    .label("名前")
    .required("名前必須です")
    .test(
      "between",
      () => "名前は2文字以上、10文字以内にしてください",
      (val) => {
        return val?.length >= 2 && val?.length <= 10;
      }
    ),
});
```

さらに上記のような独自バリエーションをプロジェクト内で汎用的に使用したい場合は、以下のように addMethod を使って登録することができます。

```tsx
yup.addMethod(yup.string, "between", function () {
  return this.test(
    "between",
    () => "名前は2文字以上、10文字以内にしてください",
    (val) => {
      return val ? val?.length >= 2 && val?.length <= 10 : false;
    }
  );
});
const schema = yup.object({
  name: yup.string().label("名前").required("名前必須です").between(),
});
```

addMethod を使用している部分を別途 ts ファイルに切り出して、addMethod を設定したモジュールを export すれば様々なコンポーネントで使用することができます。
ただし、この addMethod 周りは型定義で少し癖があります。
先程記載したコードは VsCode で記載すると、between メソッドはないというエラーが発生します。
ただ、エラーが起きたまま画面でバリデーションさせると意図した通りの動きをします。
これは addMethod で定義したバリエーションは型定義上存在しないことが原因です。
そのため、typescript で別途 addMethod で追加したバリエーションを補完で表示させるには、global.d.ts などの型定義ファイルを作成し、以下のようなコードを記載する必要があります。

```tsx
import * as yup from "yup";
declare module "yup" {
  interface StringSchema {
    between(): yup.NumberSchema<string>;
  }
}
```

こうすれば、補完で between メソッドが表示されるようになります。
ただ、正直少し使いにくいなとは思いました。
汎用的な独自型を使いこなすのはもう少し内部の理解が必要そうです。
ただ、使いこなせたら独自のバリデーションライブラリがのようなものができそうなので、複雑なフォーム操作に対応するには良さそうです。
また、その他にも出力する値に対してトリムするなど様々な加工を行うことができるので、Yup でないとしても何かしらの検証ライブラリを使う方が良いと思っています。

## おわりに

今回は React Hook Form について、簡単に見ていきました。
使うだけなら、とても簡単に導入ができるので、多くの人に使用されるだけはあるなと思いました。
Yup などの外部モジュールとの連携もとても簡単なので、検証が絡むフォーム操作を行う場合はとりあえず導入するのが良さそうです。
少し心残りなのは、formState の「is~」系がどのタイミングで切り替わるかがハッキリと分からなかったところです。
もし、ご存知の方がいればコメントで教えていただければ幸いです。
ここまで読んでいただきありがとうございました。

## 参考資料

[https://github.com/jquense/yup/issues/312](https://github.com/jquense/yup/issues/312)
