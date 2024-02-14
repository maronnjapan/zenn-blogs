---
title: "ドラックアンドドロップで要素を作成したり、並べ替えたり"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React"]
published: true
---

## はじめに

ファイルのアップロードの時しかり、TODO リストの並べ替えしかり、ドラック・ドロップする機会はそれなりにあります。
しかし、いざ自分が実装するとなるとどうして良いか分かりませんでした。
とはいえ、ここ直近でドラック・ドロップ機能を実装する必要が出てきたため調査していたら、[dnd kit](https://docs.dndkit.com/)といういい感じのモジュールがありました。
なので、今回はこの[dnd kit](https://docs.dndkit.com/)の使い方を見ていき、活用編としてドラック・ドロップで要素が生成できるようにするコンポーネントを作っていきます。
結構長いですが、是非お付き合いください。

## この記事のここだけ!!!

ドラック・ドロップ機能は[dnd kit](https://docs.dndkit.com/)で実装しています。
この機能はドラック・ドロップ機能に加え、並べ替え機能もあります。
以上の機能を活かして、ドラック・ドロップしたらコンポーネントを動的に生成できます。
生成した要素の並べ替えもできます。

## 完成品

この記事でできるのは以下のものとなっています。
![newcreateElmDragAndDrop.gif](/images/drag-and-drop-by-dnd-kit/newcreateElmDragAndDrop.gif)
コードの全体像は以下の通りです。
ドラック・ドロップについて、全体的な管理を行うコンポーネント

```jsx
import { DndContext, DragEndEvent, DragOverEvent, DragStartEvent, Modifier } from "@dnd-kit/core";
import { useState } from "react";
import DropComponent from "./DropComponent";
import { arrayMove } from "@dnd-kit/sortable";
import { v4 as uuidv4 } from 'uuid'
import { restrictToVerticalAxis } from "@dnd-kit/modifiers";
import DragComponent from "./DragComponent";
export default function DragDropContext() {
    // ドロップできる領域に表示される要素のデータ用のState
    const [elms, setElms] = useState<{ id: string, name: string }[]>([])
    // ドラック中の要素データ用のState
    const [activeElm, setActiveElm] = useState<{ id: string, name: string, virtualId?: string }>()
    // ドラックする要素がドロップ領域のものかを判定するState
    const [isDropContent, setIsDropContent] = useState(false)
    // ドラックする方向の制約を管理するState
    // 基本的に「@dnd-kit/modifiers」モジュール内の値を付与する
    const [modifiers, setModifiers] = useState<Modifier[]>([])
    // ドラッグ開始時に発火する関数
    const handleDragStart = (event: DragStartEvent) => {
        // ドラックした要素に関わるイベントを取得
        const { active } = event;
        // ドラックした要素がドロップ領域に存在するかを判定する
        const isExistInDropContent = elms.map((i) => i.id).includes(active.id.toString());
        //ドラッグした要素のid作成。ドロップ領域に存在しない場合は、新規でidを作成する
        const id = isExistInDropContent ? active.id.toString() : uuidv4();
        // ドラック中にドロップ領域内で仮生成される要素のID。ドロップ領域内のものをドラックしているときはundefined
        const virtualId = isExistInDropContent ? undefined : uuidv4();
        setActiveElm({ id, name: active.data.current?.name, virtualId })
        setIsDropContent(isExistInDropContent)
        // ドラックした要素がドロップ領域内のものの場合、動かせる方向を垂直方向のみに制限する
        isExistInDropContent ? setModifiers([restrictToVerticalAxis]) : setModifiers([])
    };
    // ドラック終了時に発火する関数
    const handleDragEnd = (e: DragEndEvent) => {
        // 値をリセットする。
        // Stateのset関数は非同期なので、ここで値をリセットしても関数内の処理には影響ない。
        setActiveElm(undefined)
        // ドロップ領域のイベント(over)を取得。
        const { over } = e;
        // 何もドラックしていない場合や、ドロップ領域外でドラックを辞めた場合処理を中断する。
        if (!over || !activeElm) {
            setElms(() => elms.filter((elm) => elm.id !== activeElm?.virtualId))
            return
        };
        // 仮要素を実際の要素として、ドロップ領域内へ反映させている
        setElms(() => elms.map((elm) => ({
            id: elm.id === activeElm.virtualId ? activeElm.id : elm.id,
            name: elm.name
        })))
        // ドロップ領域内の要素をドラックしていた場合、ドロップ領域内の要素を並べ替える
        if (elms.some((elm) => elm.id === activeElm.id)) {
            setElms((items) => {
                const oldIndex = items.map((i) => i.id).findIndex((val) => val === activeElm.id)
                const newIndex = items.map((i) => i.id).findIndex((val) => val === over.id)
                return arrayMove(items, oldIndex, newIndex);
            });
        }
    }
    const handleDragOver = (e: DragOverEvent) => {
        const { over } = e;
        //ドロップした場所にあった要素のid
        const overId = over?.id;
        // 何もドラックしていない場合や、ドロップ領域外でドラックした要素を動かしている場合処理を中断する。
        if (!overId || !activeElm) {
            // 生成した仮要素を削除する
            setElms(() => elms.filter((elm) => elm.id !== activeElm?.virtualId));
            return;
        };
        // ドラック要素がドロップ領域内に入ったら要素を追加する
        if (activeElm.virtualId && !elms.some((elm) => elm.id === activeElm.virtualId)) {
            setElms([...elms.slice(0, elms.length), { id: activeElm.virtualId, name: e.active.data.current?.name }])
        }
        // 仮の要素をドロップ領域内に動かしている時に、位置を変更する。
        if (elms.some((elm) => elm.id === activeElm.virtualId)) {
            setElms((items) => {
                const oldIndex = items.map((i) => i.id).findIndex((val) => val === activeElm.virtualId)
                const newIndex = items.map((i) => i.id).findIndex((val) => val === over.id)
                return arrayMove(items, oldIndex, newIndex);
            });
        }
    }
    return (
        <>
            <div style={{ display: 'flex' }}>
                <DndContext modifiers={modifiers} onDragEnd={(e) => handleDragEnd(e)} onDragOver={(e) => handleDragOver(e)} onDragStart={(e) => handleDragStart(e)}>
                    <DropComponent elms={elms} isDropContent={isDropContent} activeElmId={activeElm?.virtualId}>
                    </DropComponent>
                    <DragComponent></DragComponent>
                </DndContext>
            </div>
        </>
    )
}
```

ドロップ領域のコンポーネント

```tsx
import { UniqueIdentifier, useDroppable } from "@dnd-kit/core";
import { SortableContext, rectSortingStrategy } from "@dnd-kit/sortable";
import SortDropContent from "./SortDropContent";
type Props = {
  elms: { id: UniqueIdentifier; name: string }[];
  activeElmId?: string;
  isDropContent?: boolean;
};
export default function DropComponent({
  elms,
  activeElmId,
  isDropContent = false,
}: Props) {
  const { isOver, setNodeRef: dropRef } = useDroppable({
    id: "droppable",
  });
  const dropStyle = {
    padding: "1rem 0 0 0",
    minHeight: "100px",
    width: "200px",
    border: isOver && !isDropContent ? "3px solid blue" : `1px solid`,
  };
  return (
    <div style={{ margin: "2rem" }}>
      <div ref={dropRef} style={dropStyle}>
        {/* ドロップ領域内の要素群。SortableContextで囲むことで、並べ替えの設定ができる */}
        <SortableContext items={elms} strategy={rectSortingStrategy}>
          {elms.map((elm) => {
            return (
              <SortDropContent
                key={elm.id}
                elm={elm}
                isVirtual={elm.id === activeElmId}
              ></SortDropContent>
            );
          })}
        </SortableContext>
      </div>
    </div>
  );
}
```

ドロップ領域内で表示するコンポーネント

```tsx
import { UniqueIdentifier } from "@dnd-kit/core";
import { useSortable } from "@dnd-kit/sortable";
import { CSS } from "@dnd-kit/utilities";
type Props = {
  elm: { id: UniqueIdentifier; name: string };
  isVirtual: boolean;
};
export default function SortDropContent({ elm, isVirtual }: Props) {
  const {
    isDragging,
    // 並び替えのつまみ部分に設定するプロパティ
    setActivatorNodeRef,
    listeners,
    // DOM全体に対して設定するプロパティ
    setNodeRef,
    transform,
    transition,
  } = useSortable({ id: elm.id });
  return (
    <div
      ref={setNodeRef}
      style={{
        transform: CSS.Transform.toString(transform),
        transition,
      }}
    >
      <p
        ref={setActivatorNodeRef}
        {...listeners}
        style={{
          cursor: isDragging ? "grabbing" : "grab",
          border: "1px solid",
          margin: "0 0 1rem 0",
          // 仮で生成されている要素の場合、薄くする。
          opacity: isVirtual ? ".5" : "1",
        }}
      >
        {elm.name}
      </p>
    </div>
  );
}
```

ドラックできる要素が存在するコンポーネント

```tsx
import { useDraggable } from "@dnd-kit/core";
export default function DragComponent() {
  const { attributes, listeners, setNodeRef, transform } = useDraggable({
    id: "draggable",
    data: { name: "test" },
  });
  const {
    attributes: attributes2,
    listeners: listeners2,
    setNodeRef: setNodeRef2,
    transform: transform2,
  } = useDraggable({
    id: "draggable2",
    data: { name: "test2" },
  });
  const {
    attributes: attributes3,
    listeners: listeners3,
    setNodeRef: setNodeRef3,
    transform: transform3,
  } = useDraggable({
    id: "draggable3",
    data: { name: "test3" },
  });
  const style = transform
    ? {
        transform: `translate3d(${transform.x}px, ${transform.y}px, 0)`,
      }
    : undefined;
  const style2 = transform2
    ? {
        transform: `translate3d(${transform2.x}px, ${transform2.y}px, 0)`,
      }
    : undefined;
  const style3 = transform3
    ? {
        transform: `translate3d(${transform3.x}px, ${transform3.y}px, 0)`,
      }
    : undefined;
  return (
    <div>
      <button ref={setNodeRef} style={style} {...listeners} {...attributes}>
        テスト
      </button>
      <button ref={setNodeRef2} style={style2} {...listeners2} {...attributes2}>
        テスト2
      </button>
      <button ref={setNodeRef3} style={style3} {...listeners3} {...attributes3}>
        テスト3
      </button>
    </div>
  );
}
```

## 導入経緯

React でドラックアンドドロップを調べるとそれなりに記事は出てきます。
そして、[こちらの記事](https://tech-blog.talentio.co.jp/entry/2023/03/28/143944)を参考に以下 5 つのモジュールのダウンロード数を比較すると[react-draggable](https://github.com/react-grid-layout/react-draggable)が一番人気となっています。
![[npm trendsの検索結果](https://npmtrends.com/@dnd-kit/core-vs-clauderic/react-sortable-hoc-vs-react-beautiful-dnd-vs-react-dnd-vs-react-draggable-vs-react-sortable-hoc) から引用](/images/drag-and-drop-by-dnd-kit/2024-01-14_11h07_00.png)
[npm trends の検索結果](https://npmtrends.com/@dnd-kit/core-vs-clauderic/react-sortable-hoc-vs-react-beautiful-dnd-vs-react-dnd-vs-react-draggable-vs-react-sortable-hoc) から引用
なら、react-draggable を使って見ようと思いましたが、記事内でもあるようにメンテナンスがあまりされておらず、対応の React のバージョンも 16 と 18 系を想定しておりません。
Atlassian 製の react-beautiful-dnd も[package.json](https://github.com/atlassian/react-beautiful-dnd/blob/master/package.json#L120)を確認すると、18 系を想定していなさそうです。
**[react-sortable-hoc](https://github.com/clauderic/react-sortable-hoc)**は dnd kit の前身モジュールなので、今使用する意義はありません。
なので、[react-dnd](https://github.com/react-dnd/react-dnd)と[dnd kit](https://github.com/clauderic/dnd-kit)の二択になります。
正直どっちでも良さそうだったのですが、react-dnd は最終更新日が 2 年前とあまり更新はされていない一方で、dnd kit は 2 ヶ月前と更新がそれなりの頻度行われています。
また、2 つのモジュールで Google 検索すると体感ですが、dnd kit の方が比較的最近の記事を多く見かけます。
よって、今後もメンテナンスされる可能性が高い dnd kit の方がアップデートに対してすぐに対応してくれそうだと思い dnd kit を採用しました。

## dnd kit とは

[dnd kit](https://dndkit.com/)は React で使用できるドラック・ドロップ機能を有したモジュールです。
個人的にドキュメントを見たり、実際に使ってみて以下のメリットがあると感じました。
①10kb 以下とモジュールの容量が軽い
② スマホ画面での使用しやすさも考慮している。
③ モジュール単体でドラック・ドロップ機能が完結するので、導入がしやすい。
④ ドラック・ドロップ機能を使用するのに一つのコンテキストと二つの関数だけで可能となる
⑤ ソート機能も簡単に導入できる
⑥ アニメーションを導入しやすい
その他にも多くの機能や特徴はありますが、特に利点を感じたのは上記の通りです。
dnd kit の特徴をざっくり説明したので、実際に使ってみましょう。

## 基本的な実装

### dnd kit の導入

React が実行できるプロジェクトで、`npm install @dnd-kit/core @dnd-kit/modifiers @dnd-kit/sortable @dnd-kit/utilities`を実行します。
@dnd-kit/core はドラック・ドロップ機能を支える dnd kit の核となるモジュールです。
@dnd-kit/modifiers はドラックする方向を制御するのを今回行っているので、その対応のために導入しています。
同様に@dnd-kit/sortable はソート機能、@dnd-kit/utilities はアニメーション対応のために導入しています。
導入は以上となります。
別途設定ファイルなどを用意する必要はなく、後はコンポーネント内に処理を書くだけです。
簡単ですね。
なお、ドラック・ドロップ機能とは直接関係ないですが、表示させる要素の id を uuid にするために`npm install uuid`と`npm i --save-dev @types/uuid`を実行して、uuid モジュールをインストールしています。
それでは、コンポーネントに実装していきます。

### ドラック・ドロップ機能の準備とドラック機能の実装

ここではまず基本的な設定を行い、ドラック・ドロップができるようにしていきます。
まずは要素のドラック・ドロップ機能を使うための準備をします。
任意のファイルに以下の機能を記載します。

```tsx
import { DndContext, useDraggable } from "@dnd-kit/core";
import DraggableBase from "./DraggableBase";
export default function DndContextBase() {
  return (
    <DndContext>
      <DraggableBase></DraggableBase>
    </DndContext>
  );
}
```

DndContext は[React Context API](https://react.dev/learn/passing-data-deeply-with-context)を用いて実装されてる、ドラック・ドロップでのデータの受け渡しを行うコンポーネントとなっています。
dnd kit のドラック機能やドロップ機能を使いたい場合、使用するコンポーネントの外側を DndContext で囲む必要があります。
これでドラック・ドロップ機能が使えるようになったので、次にドラック機能を実装していきます。
DraggableBase コンポーネントファイルを以下のように記載します。

```tsx
import { useDraggable } from "@dnd-kit/core";
export default function DraggableBase() {
    const {  listeners transform } = useDraggable({
        id: 'draggableTest',
        data: { name: 'test' }
    });
    const style = transform ? {
        transform: `translate3d(${transform.x}px, ${transform.y}px, 0)`,
    } : undefined;
    return (
        <button style={style}  {...listeners} >
            テスト
        </button>
    )
}
```

ドラック機能を使うには[useDraggable](https://docs.dndkit.com/api-documentation/draggable/usedraggable)関数を使用します。
useDraggable 関数を呼び出すのに必要なのは、要素を一意に特定する id プロパティです。
これを設定すれば、ドラック機能を使うことができます。
ただ、この後ドラックしている要素の値を使いコンポーネントを生成したいので、要素が持つデータを定義する data プロパティも設定しています。
useDraggable 関数の戻り値についてですが、ドラック機能を開始するのに必要なものは`listeners`だけで問題ありません。
`listeners`については、useDraggable 関数で設定したドラックの検知を行いたい要素に付与する必要があります。
ただ、このままでは動かす設定はできたとしても画面上動いているようにはできません。
そこで、transform の値を使用します。
transform はドキュメントを参照すると、以下のような画面上での要素の座標を取得します。
この値を CSS に設定することで要素を動かすことができます。
ここまでできたので、一旦動かしてみると以下のようにドラックができます。
![ReactDnDAnimation.gif](/images/drag-and-drop-by-dnd-kit/ReactDnDAnimation.gif)
動かすだけならこれでいいのですが、今後ドロップ機能を使うことを考えると以下のように修正します。

```tsx
import { useDraggable } from "@dnd-kit/core";
export default function DraggableBase() {
  const { listeners, setNodeRef, transform } = useDraggable({
    id: "draggableTest",
    data: { name: "test" },
  });
  const style = transform
    ? {
        transform: `translate3d(${transform.x}px, ${transform.y}px, 0)`,
      }
    : undefined;
  return (
    <button ref={setNodeRef} style={style} {...listeners}>
      テスト
    </button>
  );
}
```

着目すべき点は ref 属性に`setNodeRef`という値を設定していることです。
これがないと、他の要素と衝突したのかなどを検知できず、この後実装したい機能に影響を及ぼします。
そのため、ドロップ機能実装前に`setNodeRef`だけは付与しておきます。

### ドロップ機能

要素をドラックできるようになったので、次にドロップ機能を実装します。
以下のようなコンポーネントファイルを作成します。

```tsx
import { useDroppable } from "@dnd-kit/core";
export default function DrppabbleBase() {
  const { isOver, setNodeRef } = useDroppable({
    id: "droppableTest",
  });
  const dropStyle = {
    marginTop: "1rem",
    padding: "1rem 0 0 0",
    minHeight: "100px",
    width: "200px",
    border: isOver ? "3px solid blue" : `1px solid`,
  };
  return <div style={dropStyle} ref={setNodeRef}></div>;
}
```

注目すべきは useDroppable 関数です。
この関数が設定されているコンポーネントはドロップ機能を有することができます。
useDroppable 関数の引数には、ドロップ受け入れ要素を一意に特定する ID が必要です。
ID は必須ですが、今回はドロップ機能のデータを使用しないので、data プロパティは無しにします。
戻り値としては、`isOver`と`setNodeRef`を取得しています。
setNodeRef はドラック要素がドロップ領域内に入ってきたことを検知するために使うので、取得が基本必須の値です。
isOver は要素がドロップ領域上でドラックされている場合、true になります。
今回はドラック要素がドロップ領域上にいる時に、スタイルの適用をしたいため取得しました。
ドロップコンポーネントも作成できたので、親側で呼び出すようにします。

```tsx
import { DndContext, useDraggable } from "@dnd-kit/core";
import DraggableBase from "./DraggableBase";
import DrppabbleBase from "./DropabbleBase";
export default function DndContextBase() {
  return (
    <DndContext>
      <DraggableBase></DraggableBase>
      <DrppabbleBase></DrppabbleBase>
    </DndContext>
  );
}
```

ここまで行うと以下のように、ドラック要素がドロップ領域上にいる場合枠が青色になります。
![ReactDnDAnimationDropOnly.gif](/images/drag-and-drop-by-dnd-kit/ReactDnDAnimationDropOnly.gif)

### ドラック要素がドロップされたときに、ドロップ領域に要素が生成されるようにする

先程の二つで、基本的なドラックドロップ機能を実装することができました。
なので、ここでは少し発展系としてドラックしている要素をドロップ領域内でドロップしたら要素が生成されるようにします。
最初に完成したコードを示します。
まずは DndContext を呼び出している、DndContextBase 関数についてです。

```jsx
import { DndContext, DragEndEvent } from "@dnd-kit/core";
import DraggableBase from "./DraggableBase";
import DrppabbleBase from "./DropabbleBase";
import { useState } from "react";
import { v4 as uuidv4 } from 'uuid'
export default function DndContextBase() {
    const [items, setItems] = useState<{ id: string, name: string }[]>([])
    const handleDragEnd = (event: DragEndEvent) => {
        if (event.over) {
            setItems((prev) => [...prev, { id: uuidv4(), name: event.active.data.current?.name }])
        }
    }
    return (
        <DndContext onDragEnd={handleDragEnd}>
            <DraggableBase></DraggableBase>
            <DrppabbleBase items={items}></DrppabbleBase>
        </DndContext>
    )
}
```

次に、ドロップ領域を示す DrppabbleBase 関数内のコードです。

```jsx
import { useDroppable } from "@dnd-kit/core";
type Props = {
  items: { id: string, name: string }[],
};
export default function DrppabbleBase({ items }: Props) {
  const { isOver, setNodeRef } = useDroppable({
    id: "droppableTest",
  });
  const dropStyle = {
    marginTop: "1rem",
    padding: "1rem 0 0 0",
    minHeight: "100px",
    width: "200px",
    border: isOver ? "3px solid blue" : `1px solid`,
  };
  return (
    <div style={dropStyle} ref={setNodeRef}>
      {items.map((item) => (
        <div style={{ marginBottom: ".5rem" }} key={item.id}>
          <button>{item.name}</button>
        </div>
      ))}
    </div>
  );
}
```

では解説していきます。
今回の話において、もっとも重要なのが以下の関数部分です。

```jsx
const handleDragEnd = (event: DragEndEvent) => {
  if (event.over) {
    setItems((prev) => [
      ...prev,
      { id: uuidv4(), name: event.active.data.current?.name },
    ]);
  }
};
```

dnd kit では[ドキュメント](https://docs.dndkit.com/api-documentation/context-provider#event-handlers)を確認すると、大きく分けて以下 4 つのイベントハンドラーが設定されています。
①onDragStart : ドラック可能要素がドラックされた時に実行します。
②onDragMove : ドラック要素を動かしいるときに発火します。
③onDragOver : ドラック要素がドロップ可能領域の上に来た時に発火します。
④onDragEnd : ドラック要素をドロップした時に発火します。
今回はドラック要素をドロップ可能領域でドロップした時に、要素を生成することが要件なので ④ の onDragEnd ハンドラーを使用します。
そのための関数が先程掲載した handleDragEnd 関数となっています。
この関数の中身をもう少しみていきます。
引数の DragEndEvent 型は以下のプロパティを有しています。

```jsx
interface DragEvent {
  activatorEvent: Event;
  active: Active;
  collisions: Collision[] | null;
  delta: Translate;
  over: Over | null;
}
export interface DragEndEvent extends DragEvent {}
```

そして、今回使用しているのは active プロパティと over プロパティです。
active プロパティはドラックしていた要素の情報を持っています。
そして、over プロパティはドラック要素をドロップした時に、ドロップ領域上であればドロップ領域の情報を持ち、そうでなければ null が設定されます。
今の説明から、以下の条件分岐している理由がドロップ領域でドラック要素を離した時のみ、State に値を追加するようするためと推測できます。

```jsx
if (event.over) {
  setItems((prev) => [
    ...prev,
    { id: uuidv4(), name: event.active.data.current?.name },
  ]);
}
```

実際に、上記分岐を無くした動きが以下のようになります。
![createAllAreaDronContent.gif](/images/drag-and-drop-by-dnd-kit/createAllAreaDronContent.gif)
ドロップ領域で離していないのに、ドロップ領域内に要素が生成されてしまいます。
これを防ぐために、ドロップ領域内でドロップした時のみ処理を行うようにしています。
そして、ドロップ領域内でドロップされたときは、ドラックしていた要素の情報を持つ active プロパティから情報を取得して、id を生成しつつ State に追加しています。
なお、active プロパティの`data.current`部分ですが、ここは useDraggable 関数を呼び出す時に設定した data プロパティ部分の値が格納されています。
今回は以下のように定義していたので、name を記載して情報を取得しています。

```jsx
useDraggable({
  id: "draggableTest",
  data: { name: "test" },
});
```

handleDragEnd 関数の中身は見たので、後はこの関数を DndContext の onDragEnd 属性に付与します。
後は、dnd kit 固有の話ではありません。
State をドロップ領域のコンポーネントに渡し、ドロップ領域内で map 関数を使いつつ要素を生成しています。
以上を行えば、ドロップ領域内でドラック要素を離したら要素が生成されるようになります。
![createElmInDropContent.gif](/images/drag-and-drop-by-dnd-kit/createElmInDropContent.gif)
ドロップ領域内でドラック要素を離した時のみ要素が生成されていることが確認できますね。

### 余談 ドラック要素の data 属性を型安全にする

先程要素を生成する内容について見ていましたが、実はドラック要素の情報を取得していた`event.active.data.current?.name`の name 部分は補完が効きません。
そのため、先程の例では自分で useDraggable 関数で設定した値を確認し、その値を手入力しています。
これはミスが起きやすく、Typescript を使っているメリットを活かせていません。
そこで、[こちらのイシュー](https://github.com/clauderic/dnd-kit/issues/935)を参考にしつつジェネリクスを使い拡張できるようにしたのが以下のコードとなります。

```jsx
import {
    Active,
    Collision,
    Data,
    DndContextProps,
    Over,
    Translate,
    UseDraggableArguments,
    UseDroppableArguments,
    useDraggable as useOriginalDraggable,
    useDroppable as useOriginalDroppable,
} from "@dnd-kit/core";
export interface UseDroppableTypesafeArguments<DRO> extends Omit<UseDroppableArguments, "data"> {
    data: DRO;
}
export function useBaseDroppable<DRO extends Data>(props: UseDroppableTypesafeArguments<DRO>) {
    return useOriginalDroppable(props);
}
export interface UseDraggableTypesafeArguments<DRA> extends Omit<UseDraggableArguments, "data"> {
    data: DRA;
}
export function useBaseDraggable<DRA extends Data>(props: UseDraggableTypesafeArguments<DRA>) {
    return useOriginalDraggable(props);
}
interface TypesafeActive<DRA> extends Omit<Active, "data"> {
    data: React.MutableRefObject<DRA | undefined>;
}
interface TypesafeOver<DRO> extends Omit<Over, "data"> {
    data: React.MutableRefObject<DRO | undefined>;
}
interface DragEvent<DRA, DRO> {
    activatorEvent: Event;
    active: TypesafeActive<DRA>;
    collisions: Collision[] | null;
    delta: Translate;
    over: TypesafeOver<DRO> | null;
}
export interface DragStartEvent<DRA, DRO> extends Pick<DragEvent<DRA, DRO>, "active"> { }
export interface DragMoveEvent<DRA, DRO> extends DragEvent<DRA, DRO> { }
export interface DragOverEvent<DRA, DRO> extends DragMoveEvent<DRA, DRO> { }
export interface DragEndEvent<DRA, DRO> extends DragEvent<DRA, DRO> { }
export interface DragCancelEvent<DRA, DRO> extends DragEndEvent<DRA, DRO> { }
export interface DndContextTypesafeProps<DRA, DRO>
    extends Omit<
        DndContextProps,
        "onDragStart" | "onDragMove" | "onDragOver" | "onDragEnd" | "onDragCancel"
    > {
    onDragStart?(event: DragStartEvent<DRA, DRO>): void;
    onDragMove?(event: DragMoveEvent<DRA, DRO>): void;
    onDragOver?(event: DragOverEvent<DRA, DRO>): void;
    onDragEnd?(event: DragEndEvent<DRA, DRO>): void;
    onDragCancel?(event: DragCancelEvent<DRA, DRO>): void;
}
```

結構色々書いていますが、やっていることは Omit で data プロパティを取り除き、型がジェネリクスである data プロパティを自前で追加しているだけです。
これによって、DndContext 内の各種イベントは設定した型が補完ができますし、useDraggable 関数、useDroppable 関数で設定する data 属性もずれが起きにくくなります。
では、実際に使ってみます。
任意のファイルに以下のようなコードを記載します。

```jsx
import {
  UseDraggableTypesafeArguments,
  useBaseDraggable,
  DndContextTypesafeProps,
} from "./wrap-dnd-kit";
type BtnData = { name: string };
export interface BtnDndContextTypesafeProps
  extends DndContextTypesafeProps<BtnData, {}> {}
export const useBtnDraggable = (
  props: UseDraggableTypesafeArguments<BtnData>
) => useBaseDraggable < BtnData > props;
```

DndCotext に渡す属性についての型と、data プロパティに型がついた useBaseDraggable 関数の拡張版を定義しています。
これを定義した後、DndContext を以下のように変更します。

```jsx
import { DndContext, DragEndEvent } from "@dnd-kit/core";
import DraggableBase from "./DraggableBase";
import DrppabbleBase from "./DropabbleBase";
import { useState } from "react";
import { v4 as uuidv4 } from 'uuid'
import { BtnDndContextTypesafeProps } from "./useBtnDndKit";
export default function DndContextBase() {
    const [items, setItems] = useState<{ id: string, name: string }[]>([])
    const contextProps: BtnDndContextTypesafeProps = {
        onDragEnd: (event) => {
            const dragName = event.active.data.current?.name;
            if (event.over && dragName) {
                setItems((prev) => [...prev, { id: uuidv4(), name: dragName }])
            }
        }
    }
    return (
        <DndContext {...contextProps}>
            <DraggableBase></DraggableBase>
            <DrppabbleBase items={items}></DrppabbleBase>
        </DndContext>
    )
}
```

DndContext に渡す Props をオブジェクトで定義した後、スプレッド構文で渡しています。
onDragEnd プロパティの中身は先程とほぼ同じですが、型がちゃんとついたことで undefined を除去して、State に保存しています。
これだけでも、型補完が効きつつ動きますが、useDraggable 関数で定義した data プロパティとずれるのは怖いので、DraggableBase 関数を以下のように変更します。

```jsx
import { useBtnDraggable } from "./useBtnDndKit";
export default function DraggableBase() {
  const { listeners, setNodeRef, transform } = useBtnDraggable({
    id: "draggableTest",
    data: { name: "test" },
  });
  const style = transform
    ? {
        transform: `translate3d(${transform.x}px, ${transform.y}px, 0)`,
      }
    : undefined;
  return (
    <button ref={setNodeRef} style={style} {...listeners}>
      テスト
    </button>
  );
}
```

useBtnDraggable 関数を使うことで、data プロパティは`{name:string}`の型でしか定義出来なくなります。
これで型の補完が効きつつ、型のずれが起きにくくなります。
もうちょい改善したい部分はありますが、`event.active.data.current`に型がつけれたのは大きいです。
今後、複数のドラック・ドロップ機能を使う場合、存在しないプロパティを定義に活用して行けたらと思います。

### ドロップ領域内の要素を並べ替えする

ドラック・ドロップで要素を生成することはできました。
しかし、ドロップ領域内に要素を生成したはいいけど、本来生成したかった場所とは違ったから、順番を入れ替えたい場合はります。
dnd kit はそういった要望にも対応しています。
なので、ここでは要素の並べ替えを実装していきます。
まずは完成したコード全体です。
おおもとの DndContextBase 関数は以下のようになります。

```jsx
import { DndContext, DragEndEvent } from "@dnd-kit/core";
import DraggableBase from "./DraggableBase";
import DrppabbleBase from "./DropabbleBase";
import { useState } from "react";
import { v4 as uuidv4 } from 'uuid'
import { BtnDndContextTypesafeProps, BtnDragEndEvent } from "./useBtnDndKit";
import { arrayMove } from "@dnd-kit/sortable";
export default function DndContextBase() {
    const [items, setItems] = useState<{ id: string, name: string }[]>([])
    const handleSort = (event: BtnDragEndEvent) => {
        const { active, over } = event;
        setItems((items) => {
            const oldIndex = items.findIndex((item) => item.id === active.id)
            const newIndex = items.findIndex((item) => item.id === over?.id)
            return arrayMove(items, oldIndex, newIndex);
        });
    }
    const addItem = (event: BtnDragEndEvent) => {
        const dragName = event.active.data.current?.name;
        if (event.over && dragName) {
            setItems((prev) => [...prev, { id: uuidv4(), name: dragName }])
        }
    }
    const contextProps: BtnDndContextTypesafeProps = {
        onDragEnd: (event) => {
            addItem(event);
            handleSort(event)
        }
    }
    return (
        <DndContext {...contextProps}>
            <DraggableBase></DraggableBase>
            <DrppabbleBase items={items}></DrppabbleBase>
        </DndContext>
    )
}
```

先程余談で作成したラップファイルに DragEndEvent に関わる型を追加します。

```jsx
import {
  UseDraggableTypesafeArguments,
  useBaseDraggable,
  DndContextTypesafeProps,
  DragEndEvent,
} from "./wrap-dnd-kit";
type BtnData = { name: string };
export interface BtnDndContextTypesafeProps
  extends DndContextTypesafeProps<BtnData, {}> {}
export interface BtnDragEndEvent extends DragEndEvent<BtnData, {}> {}
export const useBtnDraggable = (
  props: UseDraggableTypesafeArguments<BtnData>
) => useBaseDraggable < BtnData > props;
```

ドロップ領域のコンポーネントにソート可能な領域を設定します。

```jsx
import { useDroppable } from "@dnd-kit/core";
import { SortableContext } from "@dnd-kit/sortable";
import SortableBase from "./SortableBase";
export type Props = {
  items: { id: string, name: string }[],
};
export default function DrppabbleBase({ items }: Props) {
  const { isOver, setNodeRef } = useDroppable({
    id: "droppableTest",
  });
  const dropStyle = {
    marginTop: "1rem",
    padding: "1rem 0 0 0",
    minHeight: "100px",
    width: "200px",
    border: isOver ? "3px solid blue" : `1px solid`,
  };
  return (
    <div style={dropStyle} ref={setNodeRef}>
      <div
        style={{
          display: "flex",
          flexDirection: "column",
          justifyContent: "center",
          gap: ".5rem",
        }}
      >
        <SortableContext items={items}>
          {items.map((item) => (
            <SortableBase
              key={item.id}
              id={item.id}
              name={item.name}
            ></SortableBase>
          ))}
        </SortableContext>
      </div>
    </div>
  );
}
```

ソートできるよう要素を作成します。

```jsx
import { useSortable } from "@dnd-kit/sortable";
type Props = { id: string, name: string };
export default function SortableBase({ id, name }: Props) {
  const { setNodeRef, listeners } = useSortable({ id });
  return (
    <button ref={setNodeRef} {...listeners}>
      {name}
    </button>
  );
}
```

最後にソート機能とは直接関係ないですが、ソートしたことを分かりやすくするためにドラック要素を追加します。

```jsx
import { useBtnDraggable } from "./useBtnDndKit";
export default function DraggableBase() {
  const { listeners, setNodeRef, transform } = useBtnDraggable({
    id: "draggableTest",
    data: { name: "test" },
  });
  const {
    listeners: listeners2,
    setNodeRef: setNodeRef2,
    transform: transform2,
  } = useBtnDraggable({
    id: "draggableTest2",
    data: { name: "test2" },
  });
  const style = transform
    ? {
        transform: `translate3d(${transform.x}px, ${transform.y}px, 0)`,
      }
    : undefined;
  const style2 = transform2
    ? {
        transform: `translate3d(${transform2.x}px, ${transform2.y}px, 0)`,
      }
    : undefined;
  return (
    <>
      <button ref={setNodeRef} style={style} {...listeners}>
        テスト
      </button>
      <button ref={setNodeRef2} style={style2} {...listeners2}>
        テスト2
      </button>
    </>
  );
}
```

こうすれば、以下のようにソートができるようになります。
![baseSortElm.gif](/images/drag-and-drop-by-dnd-kit/baseSortElm.gif)
ではコードの中身についてみていきます。
まずはソート領域の確保です。
以下のように SortableContext コンポーネントで囲むことで、子供の要素はソート可能となります。

```jsx
<SortableContext items={items}>
  {items.map((item) => (
    <SortableBase key={item.id} id={item.id} name={item.name}></SortableBase>
  ))}
</SortableContext>
```

ただ、items 属性は必要です。
この items 属性は以下の型定義から分かるように、string もしくは number 型の配列か、id プロパティを持つオブジェクトの配列を格納します。

```jsx
export interface Props {
    items: (UniqueIdentifier | {
        id: UniqueIdentifier;
    })[]
		//...略
}
export declare type UniqueIdentifier = string | number;
```

この属性によって、ソート領域にあるどの要素が何番目に存在するかを判定するため必須のプロパティとなっています。
ソート領域は確保できたので、ソート要素について見ていきます。
ソート要素は以下のように、useSortable 関数を呼び出し、動かしたい要素に必要な属性を付与します。

```jsx
const { setNodeRef, listeners } = useSortable({ id });
return (
  <button ref={setNodeRef} {...listeners}>
    {name}
  </button>
);
```

ソート要素が識別できるように、useSortable 関数へ一意の ID を渡し、衝突判定ができる`setNodeRef`とイベントを検知するために`listeners`を付与します。
使い方自体は、大体ドラック・ドロップの時と同じです。
以上で、ソートを行う準備はできました。
ただ、これだけでソートを行うことはできません。
ソートの処理自体は DndContext コンポーネントの onDragEnd 属性で行う必要があるからです。
なので、DndContextBase 関数の onDragEnd プロパティ部分の処理を関数に分割し、以下のように変更しました。

```jsx
const handleSort = (event: BtnDragEndEvent) => {
  const { active, over } = event;
  setItems((items) => {
    const oldIndex = items.findIndex((item) => item.id === active.id);
    const newIndex = items.findIndex((item) => item.id === over?.id);
    return arrayMove(items, oldIndex, newIndex);
  });
};
const addItem = (event: BtnDragEndEvent) => {
  const dragName = event.active.data.current?.name;
  if (event.over && dragName) {
    setItems((prev) => [...prev, { id: uuidv4(), name: dragName }]);
  }
};
const contextProps: BtnDndContextTypesafeProps = {
  onDragEnd: (event) => {
    addItem(event);
    handleSort(event);
  },
};
```

ここで重要なのは arrayMove 関数です。
[ソースコード](https://github.com/clauderic/dnd-kit/blob/master/packages/sortable/src/utilities/arrayMove.ts)を確認すると、以下のようになっています。

```jsx
/**
 * Move an array item to a different position. Returns a new array with the item moved to the new position.
 */
export function arrayMove<T>(array: T[], from: number, to: number): T[] {
  const newArray = array.slice();
  newArray.splice(
    to < 0 ? newArray.length + to : to,
    0,
    newArray.splice(from, 1)[0]
  );
  return newArray;
}
```

動かしたい要素が存在する index 番号を第二引数に指定して、動かす index 番号先を第三引数に設定すると配列をいい感じに操作してくれそうです。
なので、今回はドラックしている要素が存在していた index 番号を findIndex メソッドで取得し、ドラック要素を離した時の下にいた要素の index 番号も同様に findeIndex メソッドで取得しています。

```jsx
const oldIndex = items.findIndex((item) => item.id === active.id);
const newIndex = items.findIndex((item) => item.id === over?.id);
```

なお、over プロパティですが`setNodeRef`が設定されている一番上の要素の値を取得します。
そのため以下のような要素があった場合、要素 ② の上でドラック要素を離したら over プロパティは要素 ② の値を取得します。
![setNodeRef.drawio.png](</images/drag-and-drop-by-dnd-kit/setNodeRef.drawio_(1).png>)
一方で、要素 ③ でドラック要素を離したとしても、`setNodeRef`はないため over プロパティは要素 ③ の値を取得しません。
その代わり、要素 ③ を囲っている要素 ① は`setNodeRef`が設定されているので、over プロパティは要素 ① の値を取得します。
よって、以下のコードはドラック要素がドロップした領域にいたソート可能な要素の index 番号に移動させ、該当の index 番号以降にある要素は一つずつ index 番号が増えることによって並べ替えが行われていると理解できます。

```jsx
setItems((items) => {
  const oldIndex = items.findIndex((item) => item.id === active.id);
  const newIndex = items.findIndex((item) => item.id === over?.id);
  return arrayMove(items, oldIndex, newIndex);
});
```

以上で基本的なドラック・ドロップ機能や、要素の並べ替えを行うことができました。
なので、ここからは dnd kit で提供されている他の機能も使いつつ、最初に示した以下の動きの実装を解説していきます。
![newcreateElmDragAndDrop.gif](/images/drag-and-drop-by-dnd-kit/newcreateElmDragAndDrop2.gif)

## 完成したコードの解説

### 大元部分

まずは DndContext を使っている大元から解説します。

```jsx
// ドロップできる領域に表示される要素のデータ用のState
    const [elms, setElms] = useState<{ id: string, name: string }[]>([])
    // ドラック中の要素データ用のState
    const [activeElm, setActiveElm] = useState<{ id: string, name: string, virtualId?: string }>()
    // ドラックする要素がドロップ領域のものかを判定するState
    const [isDropContent, setIsDropContent] = useState(false)
    // ドラックする方向の制約を管理するState
    // 基本的に「@dnd-kit/modifiers」モジュール内の値を付与する
    const [modifiers, setModifiers] = useState<Modifier[]>([])
```

こちらは基本的に state 定義です。
コメント通りの内容ではありますが、一点補足があります。
一番下の Modifier 型の配列を定義している State ですが、これは dnd kit で提供されている[Modifiers](https://docs.dndkit.com/api-documentation/modifiers)という機能の値が格納します。
Modifiers は凄くザックリ言うと、ドラック可能な方向を限定する値となっています。
これを DndConetxt で以下のように定義すると、内部のドラック要素の動かす方向を限定できます。

```jsx
<DndContext modifiers={modifiers}>//...略</DndContext>
```

次にドラック要素をつかんだときの処理をみていきます。

```jsx
// ドラッグ開始時に発火する関数
const handleDragStart = (event: DragStartEvent) => {
  // ドラックした要素に関わるイベントを取得
  const { active } = event;
  // ドラックした要素がドロップ領域に存在するかを判定する
  const isExistInDropContent = elms
    .map((i) => i.id)
    .includes(active.id.toString());
  //ドラッグした要素のid作成。ドロップ領域に存在しない場合は、新規でidを作成する
  const id = isExistInDropContent ? active.id.toString() : uuidv4();
  // ドラック中にドロップ領域内で仮生成される要素のID。ドロップ領域内のものをドラックしているときはundefined
  const virtualId = isExistInDropContent ? undefined : uuidv4();
  setActiveElm({ id, name: active.data.current?.name, virtualId });
  setIsDropContent(isExistInDropContent);
  // ドラックした要素がドロップ領域内のものの場合、動かせる方向を垂直方向のみに制限する
  isExistInDropContent
    ? setModifiers([restrictToVerticalAxis])
    : setModifiers([]);
};
```

処理の内容はコメント通りですが、ここではドラックしている要素を State に格納することと、ドラック要素がドロップ領域内のものかどうかで設定する値を変えています。
補足的に説明する部分は`virtualId`部分です。
この項目を設けているのは、ドラックしている状態でもドロップ領域内に要素を生成して動かせるようにはしたいけど、ドラック要素を離した時は通常の id で生成するようにしたいためです。
この`virtualId`を設けず、id だけで行うとドラック中にできた要素に加えて、ドラックを辞めた時も要素が生成されてしまいます。
それを防ぐために、最終的な要素生成までは`virtualId`で仮生成を行い、正式に生成される際は id を使うようにしています。
そのためにも`virtualId`を設けています。
次にドラック要素がドロップ領域内の要素と接触した時に実行される処理についてです。

```jsx
const handleDragOver = (e: DragOverEvent) => {
  const { over } = e;
  //ドロップした場所にあった要素のid
  const overId = over?.id;
  // 何もドラックしていない場合や、ドロップ領域外でドラックした要素を動かしている場合処理を中断する。
  if (!overId || !activeElm) {
    // 生成した仮要素を削除する
    setElms(() => elms.filter((elm) => elm.id !== activeElm?.virtualId));
    return;
  }
  // ドラック要素がドロップ領域内に入り、同じ要素が無ければ要素を追加する
  if (
    activeElm.virtualId &&
    !elms.some((elm) => elm.id === activeElm.virtualId)
  ) {
    setElms([
      ...elms.slice(0, elms.length),
      { id: activeElm.virtualId, name: e.active.data.current?.name },
    ]);
  }
  // 仮の要素をドロップ領域内に動かしている時に、位置を変更する。
  if (elms.some((elm) => elm.id === activeElm.virtualId)) {
    setElms((items) => {
      const oldIndex = items
        .map((i) => i.id)
        .findIndex((val) => val === activeElm.virtualId);
      const newIndex = items
        .map((i) => i.id)
        .findIndex((val) => val === over.id);
      // 要素を指定の場所に並び替える
      return arrayMove(items, oldIndex, newIndex);
    });
  }
};
```

ここはドラックしている際に要素を仮生成する部分と、位置を変更する処理を行っています。
具体的には以下の動作部分です。
![onlyVirtualElmSor.gif](/images/drag-and-drop-by-dnd-kit/onlyVirtualElmSor.gif)
イベントハンドラーの最後である、ドラック要素を離した時の処理についても確認します。

```jsx
// ドラック終了時に発火する関数
const handleDragEnd = (e: DragEndEvent) => {
  // 値をリセットする。
  // Stateのset関数は非同期なので、ここで値をリセットしても関数内の処理には影響ない。
  setActiveElm(undefined);
  // ドロップ領域のイベント(over)を取得。
  const { over } = e;
  // 何もドラックしていない場合や、ドロップ領域外でドラックを辞めた場合処理を中断する。
  if (!over || !activeElm) {
    // 仮要素が生成されていたときは削除する
    setElms(() => elms.filter((elm) => elm.id !== activeElm?.virtualId));
    return;
  }
  // 仮要素を実際の要素として、ドロップ領域内へ反映させている
  setElms(() =>
    elms.map((elm) => ({
      id: elm.id === activeElm.virtualId ? activeElm.id : elm.id,
      name: elm.name,
    }))
  );
  // ドロップ領域内の要素をドラックしていた場合、ドロップ領域内の要素を並べ替える
  if (elms.some((elm) => elm.id === activeElm.id)) {
    setElms((items) => {
      const oldIndex = items
        .map((i) => i.id)
        .findIndex((val) => val === activeElm.id);
      const newIndex = items
        .map((i) => i.id)
        .findIndex((val) => val === over.id);
      return arrayMove(items, oldIndex, newIndex);
    });
  }
};
```

最後の部分に似た処理がありますが、これはドロップ領域内をソートした時に実際移動できるようにするためです。
ドロップ領域内の予想と接触した時の処理で記載していたのは、仮要素を並べ替えるためでしたがここでは以下のように生成されている要素を並べ替えるために記載しています。
![onlyVirtualElmSort2.gif](/images/drag-and-drop-by-dnd-kit/onlyVirtualElmSort2.gif)
以上が DndContext を呼び出している部分の主要な機能です。
次にドラック要素を記載したコンポーネントの解説といきたいところですが、基本的な実装の時に記載した以上のものはないので、省略します。

### ドロップ領域のコンポーネント

ドロップ領域のコンポーネントですが、ここも基本的な実装で解説した内容が多いので、あまり解説することはありません。
ただ、以下のスタイル定義についてだけ補足します。「

```jsx
const dropStyle = {
  padding: "1rem 0 0 0",
  minHeight: "100px",
  width: "200px",
  border: isOver && !isDropContent ? "3px solid blue" : `1px solid`,
};
```

border の部分ですが、以下のようにドロップ領域外の要素がドロップ領域内に入ってきた時のみ青枠を表示するようにしています。
![showBorder.gif](/images/drag-and-drop-by-dnd-kit/showBorder.gif)
これによって、ドロップ領域内に入ったことが分かりやすいかなと思い設定しました。
ただ、上記条件以外で青枠になってしまうと操作の邪魔になると思い、それ以外の場合では特に枠線の変更をしないようにしています。

### ソート可能な要素

最後にソート要素についてのコンポーネントです。
基本的な機能についてはすでに言及していますが、以下の部分は補足します。

```jsx
import { CSS } from "@dnd-kit/utilities";
//...略
const {
  //...略
  transform,
  transition,
} = useSortable({ id: elm.id });
return (
  <div
    style={{
      transform: CSS.Transform.toString(transform),
      transition,
    }}
  >
    //...略
  </div>
);
```

基本的な実装では並べ替え自体はできていましたが、変更する際のアニメーションがなく、初見では並べ替えが行えているのか分かりにくい状態です。
上記の状態を解消するために、アニメーションを付与することが解消方法の一つとなります。
このアニメーションを付与するのが、`transform`と`transition`になります。
これらの値を用いることで、並べ替えにアニメーションが付与され、現在の並べ替えの状態が分かりやすいものとなります。
なお、位置変更を行う transform は useSortable 関数の値をそのまま付与しても上手く動きません。
以下のように座標についての値は保持しているのです、transform プロパティ用の形になっていません。

```jsx
export declare type Transform = {
    x: number;
    y: number;
    scaleX: number;
    scaleY: number;
};
```

そこで、`@dnd-kit/utilities`の CSS オブジェクトの toString メソッドを用いることで、取得した座標をいい感じに transform プロパティで動くような値に変換してくれます。

## おわりに

今回は dnd kit を用いたドラック・ドロップ機能の解説とドラック・ドロップ機能を使った要素生成のコードの解説を行いました。
結構ボリュームが大きくなってしまいましたが、基本的な機能について記載できたので満足です。
ただ、今回実装したのはあくまで基本的な動作のみとなっています。
まだまだ機能あり、活用できる場面はあるとおもいますので気になる方は是非とも試してみてください。
ここまで読んでいただきありがとうございました。
