---
title: "Lexicalの開発記録：メンション周りの機能搭載"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Lexical", "React"]
published: true
---

# はじめに

現在[Lexical](https://lexical.dev/)を用いてエディタ開発をしています。
そこで、先週行った機能開発などについてふれていきます。
良ければ見て行ってください。
なお、ちょいちょい実際に動かすには必要な手順を省略しています。
環境構築周りはできていると仮定の上で記載しています。
よろしくお願いします。

# メンション機能が要素をホバーした時に要素が表示されるようにした

## デモ

![mention-demo.gif](/images/development-lexical-editor-2024-07-01/mention-demo.gif)

## **Lexical で設定する Plugin 部分**

### 全体的なコード

```tsx
import { useLexicalComposerContext } from "@lexical/react/LexicalComposerContext";
import {
  $createTextNode,
  $getSelection,
  $insertNodes,
  $isRangeSelection,
  $isTextNode,
  TextNode,
  KEY_ESCAPE_COMMAND,
  COMMAND_PRIORITY_HIGH,
} from "lexical";
import { $isLinkNode } from "@lexical/link";
import { mergeRegister } from "@lexical/utils";
import { useEffect, useState } from "react";
import { getSelectedNode } from "../../utils/getSelectedNode";
import {
  $createDisableMentionNode,
  $createMentionNode,
  $isDisableMentionNode,
  $isMentionNode,
} from "./node";
import SelectsFloating from "./SelectsFloating";
import HoverInfo from "./HoverInfo";
export default function MentionPlugin({
  anchorElm,
}: {
  anchorElm?: HTMLElement | null;
}) {
  const [editor] = useLexicalComposerContext();
  const [mentionText, setMentionText] = useState("");
  const [mentionElm, setMentionElm] = useState<HTMLElement | null>(null);
  useEffect(() => {
    /** 複数のイベントをまとめて登録する */
    return (
      mergeRegister(
        /** エディタが更新されたタイミングで発火させる */
        editor.registerUpdateListener(() => {
          editor.update(() => {
            /** 現在選択しているテキストの情報やNodeの情報を持つSelectionオブジェクトを取得 */
            const selection = $getSelection();
            /** selectionオブジェクトが存在しない = 特にエディタで入力などの操作をしていない　場合は何もしない */
            if (!$isRangeSelection(selection)) {
              return null;
            }
            /** 操作対象となっているNodeを取得する */
            const node = getSelectedNode(selection);
            /** MentionNodeはElementNodeを継承しているため、テキスト部分はTextNodeである
             * そのため、MentionNode部分を取得するために、対象Nodeの親Nodeを取得するようにしている
             */
            const parent = node.getParent();
            /** Node内のテキストを取得する */
            const text = node.getTextContent();
            /** キャレットの左隣にある単語を取得 */
            const nearestText = text.substring(
              selection.anchor.offset - 1,
              selection.anchor.offset
            );
            /** 同一行にあり、キャレットより二つ以上左にある単語を取得 */
            const prevText = text.substring(0, selection.anchor.offset - 1);
            /** 同一行にあり、キャレットより右にある単語を取得 */
            const afterText = text.substring(selection.anchor.offset + 1);
            if ($isMentionNode(parent)) {
              /** MentionNode内で末尾に二個空白をあけたら新しくTextNodeを後ろに作成し、そこに移動するようにした */
              const escapePattern = /\s\s$/gi;
              if (escapePattern.test(node.getTextContent())) {
                parent.insertAfter($createTextNode(" "));
                const replaceNode = $createTextNode(
                  node.getTextContent().trim()
                );
                node.replace(replaceNode);
                parent.selectNext();
                return;
              }
              /** 対象のMentionNodeのHTML要素を取得し、
               * @を除いたテキストとHTML要素をStateにセットした
               * */
              const mentionElmByKey = editor.getElementByKey(parent.getKey());
              const searchText = node.getTextContent().substring(1);
              setMentionElm(() => mentionElmByKey);
              setMentionText(() => searchText);
              return;
            }
            /** ESCを押して、メンション機能を取り消した要素の場合何もしない */
            if ($isDisableMentionNode(parent)) {
              return;
            }
            /** 単純なテキストで@を入力した時、メンション機能を有効化させる */
            if (
              nearestText === "@" &&
              $isTextNode(node) &&
              !$isLinkNode(parent)
            ) {
              /** @の前後をそれぞれ通常のテキストに変更する */
              /** LinkNodeなど特殊なテキストについては対象にならないので、影響はない認識 */
              const prevTextNode =
                prevText.length > 0 ? $createTextNode(prevText) : null;
              const afterTextNode =
                afterText.length > 0 ? $createTextNode(afterText) : null;
              /** @部分をMentionNodeとする */
              const mentionNode = $createMentionNode().append(
                $createTextNode("@")
              );
              const insertNodes = [
                prevTextNode,
                mentionNode,
                afterTextNode,
              ].filter((n): n is TextNode => n !== null);
              /** 既存Nodeを削除し、上記で構築したNode群を挿入する */
              node.remove();
              $insertNodes(insertNodes);
              /** @の後ろにキャレットを持っていく */
              mentionNode.select(1);
              return;
            }
            setMentionElm(() => null);
          });
        })
      ),
      /** ESCキーが押されたタイミングでイベントを実行する */
      editor.registerCommand(
        KEY_ESCAPE_COMMAND,
        () => {
          const selection = $getSelection();
          if (!$isRangeSelection(selection)) {
            return false;
          }
          const node = getSelectedNode(selection);
          const parent = node.getParent();
          if ($isMentionNode(parent)) {
            /** MentionNodeでESCキーが押されたら、メンション機能を無効化したNodeに置き換える */
            const disableMentionNode = $createDisableMentionNode().append(
              $createTextNode(parent.getTextContent())
            );
            parent.insertAfter(disableMentionNode);
            parent.remove();
            disableMentionNode.selectEnd();
            setMentionElm(() => null);
            return true;
          }
          return false;
        },
        COMMAND_PRIORITY_HIGH
      )
    );
  }, [editor]);
  return (
    <>
      {!!anchorElm && !!mentionElm && (
        <SelectsFloating
          anchorElm={anchorElm}
          searchText={mentionText}
          mentionElm={mentionElm}
        ></SelectsFloating>
      )}
      {!!anchorElm && <HoverInfo anchorElm={anchorElm}></HoverInfo>}
    </>
  );
}
```

基本的にはメンション表示する時の制御と、メンション機能を取り消す時の挙動を制御しています。

### メンション機能を登録したりする部分

editor.registerUpdateListener で、メンションを表示する時のイベントを設定しています。
以下部分で、メンション Node にする部分とその中で持っているテキストの設定をおこなっています。

```tsx
/** 現在選択しているテキストの情報やNodeの情報を持つSelectionオブジェクトを取得 */
const selection = $getSelection();
/** selectionオブジェクトが存在しない = 特にエディタで入力などの操作をしていない　場合は何もしない */
if (!$isRangeSelection(selection)) {
  return null;
}
/** 操作対象となっているNodeを取得する */
const node = getSelectedNode(selection);
/** MentionNodeはElementNodeを継承しているため、テキスト部分はTextNodeである
 * そのため、MentionNode部分を取得するために、対象Nodeの親Nodeを取得するようにしている
 */
const parent = node.getParent();
/** Node内のテキストを取得する */
const text = node.getTextContent();
/** キャレットの左隣にある単語を取得 */
const nearestText = text.substring(
  selection.anchor.offset - 1,
  selection.anchor.offset
);
/** 同一行にあり、キャレットより二つ以上左にある単語を取得 */
const prevText = text.substring(0, selection.anchor.offset - 1);
/** 同一行にあり、キャレットより右にある単語を取得 */
const afterText = text.substring(selection.anchor.offset + 1);
```

`@`をメンション機能を始めるための文字列としたいので、@と@より前と後ろの文字列をそれぞれ予めとっておきます。
後はそれぞれ以下のような分岐で処理を行います。

```tsx
/** すでにメンション機能を持つテキストの上にいる場合
 * この場合は、メンション内で検索ができるようにすることと
 * スペース二回おしなどで、メンション機能の後ろにもテキストを入力できる機能を実装
 */
if ($isMentionNode(parent)) {
  /** MentionNode内で末尾に二個空白をあけたら新しくTextNodeを後ろに作成し、そこに移動するようにした */
  const escapePattern = /\s\s$/gi;
  if (escapePattern.test(node.getTextContent())) {
    return;
  }
  return;
}
/** ESCを押して、メンション機能を取り消した要素の場合何もしない */
if ($isDisableMentionNode(parent)) {
  return;
}
/** 単純なテキストで@を入力した時、メンション機能を有効化させる */
if (nearestText === "@" && $isTextNode(node) && !$isLinkNode(parent)) {
  return;
}
```

それぞれ以下の分岐があります。
① すでにメンション機能の場合は、メンションから抜け出すか検索用のテキストを取得する
② メンション機能を取り消している場合は、何もしない
③ 通常のテキストで@を入力した場合、メンション機能に変換させる
各内部については結構ゴリゴリ書いていますが、達成したいことは上記の通りです。

### メンション機能の解消

Esc キーを入力したときの挙動について、イベントを登録しておきます。

```tsx
/** ESCキーが押されたタイミングでイベントを実行する */
editor.registerCommand(
  KEY_ESCAPE_COMMAND,
  () => {
    const selection = $getSelection();
    if (!$isRangeSelection(selection)) {
      return false;
    }
    const node = getSelectedNode(selection);
    const parent = node.getParent();
    if ($isMentionNode(parent)) {
      /** MentionNodeでESCキーが押されたら、メンション機能を無効化したNodeに置き換える */
      const disableMentionNode = $createDisableMentionNode().append(
        $createTextNode(parent.getTextContent())
      );
      parent.insertAfter(disableMentionNode);
      parent.remove();
      disableMentionNode.selectEnd();
      setMentionElm(() => null);
      return true;
    }
    return false;
  },
  COMMAND_PRIORITY_HIGH
);
```

Lexical では editor.registerCommand で予め特定の操作に対しての処理を登録できます。
基本的には`createCommadn`という関数を使い、キーとなる操作を登録するのですが、`KEY_ESCAPE_COMMAND`については予め Lexical 側で操作が登録されているので、そのまま使用します。
後は、実行した時の Selection オブジェクト(≒ キャレットがいる Node 情報などを持つ)がメンション Node を持っている場合、メンション機能を出現させないようにする`DisableMentionNode`へ置き換えるようにしています。
これでただの文字列のように扱うことができます。

### MentionNode と DisableMentionNode

全体的なコード

```tsx
import {
  $applyNodeReplacement,
  type DOMConversionMap,
  type DOMConversionOutput,
  type EditorConfig,
  type LexicalNode,
  type NodeKey,
  type Spread,
  ElementNode,
  SerializedElementNode,
} from "lexical";
export type SerializedMentionNode = Spread<
  { desidedText?: string },
  SerializedElementNode
>;
function $convertMentionElement(
  domNode: HTMLElement
): DOMConversionOutput | null {
  const textContent = domNode.textContent;
  if (textContent !== null) {
    const node = $createMentionNode();
    return {
      node,
    };
  }
  return null;
}
function $convertDisableMentionElement(
  domNode: HTMLElement
): DOMConversionOutput {
  const node = $createDisableMentionNode();
  return {
    node,
  };
}
export class MentionNode extends ElementNode {
  __desidedText?: string;
  static getType(): string {
    return "mention";
  }
  static clone(node: MentionNode): MentionNode {
    return new MentionNode(node.__desidedText, node.__key);
  }
  constructor(desidedText?: string, key?: NodeKey) {
    super(key);
    this.__desidedText = desidedText ?? undefined;
  }
  createDOM(config: EditorConfig): HTMLSpanElement {
    const element = document.createElement("span");
    element.setAttribute("data-lexical-mention", "true");
    element.style.color = "blue";
    return element;
  }
  updateDOM() {
    return false;
  }
  static importDOM(): DOMConversionMap | null {
    return {
      span: (node: Node) => ({
        conversion: $convertMentionElement,
        priority: 0,
      }),
    };
  }
  static importJSON(serializedNode: SerializedMentionNode): MentionNode {
    const node = $createMentionNode(serializedNode.desidedText);
    return node;
  }
  exportJSON(): SerializedMentionNode {
    return {
      ...super.exportJSON(),
      desidedText: this.getDesidedText(),
      type: "mention",
      version: 1,
    };
  }
  isInline(): boolean {
    return true;
  }
  setDesidedText(arg: string | undefined) {
    const writable = this.getWritable();
    writable.__desidedText = arg;
  }
  getDesidedText() {
    return this.__desidedText;
  }
}
export function $createMentionNode(desidedText?: string): MentionNode {
  const mentionNode = new MentionNode(desidedText);
  return $applyNodeReplacement(mentionNode);
}
export function $isMentionNode(
  node: LexicalNode | null | undefined
): node is MentionNode {
  return node instanceof MentionNode;
}
export type SerializedDisableMentionNode = SerializedElementNode;
export class DisableMentionNode extends ElementNode {
  static getType(): string {
    return "disable-mention";
  }
  static clone(node: MentionNode): DisableMentionNode {
    return new DisableMentionNode(node.__key);
  }
  updateDOM() {
    return false;
  }
  static importDOM(): DOMConversionMap | null {
    return {
      span: (node: Node) => ({
        conversion: $convertDisableMentionElement,
        priority: 0,
      }),
    };
  }
  static importJSON(
    serializedNode: SerializedDisableMentionNode
  ): DisableMentionNode {
    const node = $createDisableMentionNode();
    return node;
  }
  constructor(key?: NodeKey) {
    super(key);
  }
  createDOM(config: EditorConfig): HTMLSpanElement {
    const element = document.createElement("span");
    element.setAttribute("data-lexical-disable-mention", "true");
    return element;
  }
  exportJSON(): SerializedDisableMentionNode {
    return {
      ...super.exportJSON(),
      type: "disable-mention",
      version: 1,
    };
  }
  isInline(): true {
    return true;
  }
}
export function $createDisableMentionNode(): DisableMentionNode {
  const disableMentionNode = new DisableMentionNode();
  return $applyNodeReplacement(disableMentionNode);
}
export function $isDisableMentionNode(
  node: LexicalNode | null | undefined
): node is DisableMentionNode {
  return node instanceof DisableMentionNode;
}
```

それぞれメンション機能を持つ Node とメンション機能が出現しない Node を定義しています。
内部には深く触れませんが、基本的には以下のものを定義します。

- Node の種類を一意に特定できる Key の定義
- 画面上で表示する HTML 要素の定義
- Lexical はデータとして、Editor State というものを持つのですが、それに合わせた形で Node を情報を渡せるように JSON を定義
- Node を作成するためのユーティリティ関数の定義
- 型ガード用のユーティリティ関数の定義

以上を設定した MentionNode と DisableMentionNode をそれぞれ作成しています。
やっていることは複雑でないのですが、それぞれの戻り値の意味についてはまだまだ勉強中です。
引き続き学習し、分かり次第展開します。

## メンション要素をホバーした時上に出てくる要素

全体的なコード

```tsx
import { useLexicalComposerContext } from "@lexical/react/LexicalComposerContext";
import { $getNearestNodeFromDOMNode } from "lexical";
import { CSSProperties, useEffect, useRef, useState } from "react";
import { createPortal } from "react-dom";
import { $isMentionNode } from "./node";
let isLoading = false;
/**
 * MentionNodeをホバーした時に出現する要素
 */
export default function HoverInfo({ anchorElm }: { anchorElm: HTMLElement }) {
  const hoverInfoRef = useRef<HTMLDivElement | null>(null);
  const [editor] = useLexicalComposerContext();
  /** ホバー要素のスタイルを保持 */
  const [menuStyles, setMenuStyles] = useState<CSSProperties | null>();
  useEffect(() => {
    /** エディターの中身が更新・操作されたときに起動させる */
    return editor.registerUpdateListener(({ editorState }) => {
      /** 現在のEditor State(≒エディタ内の情報)からNode部分を抽出 */
      [...Array.from(editorState._nodeMap.values())].forEach((val) => {
        /** MentionNodeでない場合は何もしない */
        if (!$isMentionNode(val)) {
          return;
        }
        /** Nodeの種類を一意に識別するKeyからHTML要素を取得 */
        const mentionElm = editor.getElementByKey(val.getKey());
        if (!mentionElm) {
          return;
        }
        /** 既存のイベントを解除する
         * ここの解除がちゃんと意味あるかは未検証
         */
        mentionElm.removeEventListener("click", (e) => {
          showInfo(e.target);
        });
        mentionElm.removeEventListener("mouseenter", (e) => {
          showInfo(e.target);
        });
        mentionElm.removeEventListener("mouseleave", (e) => {
          if (e.relatedTarget === hoverInfoRef.current) {
            return;
          }
          setMenuStyles(() => null);
        });
        /**
         * MentionNodeを示すHTML要素に対して各種イベントハンドラーを登録する
         */
        mentionElm.addEventListener("click", (e) => {
          showInfo(e.target);
        });
        mentionElm.addEventListener("mouseenter", (e) => {
          showInfo(e.target);
        });
        mentionElm.addEventListener("mouseleave", (e) => {
          if (e.relatedTarget === hoverInfoRef.current) {
            return;
          }
          setMenuStyles(() => null);
        });
      });
    });
  }, [editor]);
  const getMentionHoberElmStyles = (
    nowMentionElm: HTMLElement
  ): CSSProperties | null => {
    if (!anchorElm || !nowMentionElm) {
      return null;
    }
    /**
     * エディタ自体の底から対象としているMentionNodeの底を引いている
     * そして、MentionNodeと被らないようにMentionNodeの高さ文上に表示するようにしている
     */
    const bottom =
      anchorElm.getBoundingClientRect().bottom -
      nowMentionElm.getBoundingClientRect().bottom +
      nowMentionElm.getBoundingClientRect().height;
    const left =
      nowMentionElm.getBoundingClientRect().left -
      anchorElm.getBoundingClientRect().left;
    return {
      bottom: `${bottom}px`,
      left: `${left}px`,
      position: "absolute",
      zIndex: 1,
      borderRadius: "8px",
      backgroundColor: "#fff",
      border: "2px solid #efefef",
      maxWidth: "200px",
      height: "fit-content",
      maxHeight: "200px",
      padding: "0.75rem 0",
      overflowY: "auto",
    };
  };
  const showInfo = (eventElm: EventTarget | null) => {
    // クリックしたリンクの要素取得
    const mentionDom = eventElm as HTMLElement;
    if (mentionDom === null) {
      return;
    }
    editor.update(() => {
      /**
       * イベント経由で取得したDOMが属しているLexicalのNodeを取得する。
       * DOM自体がLexicalのNodeとして登録されたものから生成されていれば、そのNodeを返す
       * 上記の状況でない場合は、親のNodeを探していきLexicalのNodeを返す
       * 以下ライブラリのソースコード
       * https://github.com/facebook/lexical/blob/28b3f901445d730748361e008033d876991379d3/packages/lexical/src/LexicalUtils.ts#L440
       */
      const mentionNode = $getNearestNodeFromDOMNode(mentionDom);
      if (!$isMentionNode(mentionNode)) {
        return setMenuStyles(() => null);
      }
      setMenuStyles(() => getMentionHoberElmStyles(mentionDom));
    });
  };
  return (
    <>
      {createPortal(
        <>
          {menuStyles ? (
            <div
              ref={hoverInfoRef}
              style={menuStyles}
              onMouseLeave={(e) => {
                setMenuStyles(() => null);
              }}
            >
              {isLoading ? (
                <div
                  className="flex justify-center h-full"
                  aria-label="読み込み中"
                >
                  <div className="animate-spin h-10 w-10 border-4 border-blue-500 rounded-full border-t-transparent"></div>
                </div>
              ) : (
                <div>ホバー要素</div>
              )}
            </div>
          ) : null}
        </>,
        anchorElm
      )}
    </>
  );
}
```

ホバー時にメンション要素が持っている情報が出てくるコンポーネントです。
具体的には下記の時の挙動です。
![mention-hover-deo.gif](/images/development-lexical-editor-2024-07-01/mention-hover-deo.gif)
本当は MentionNode から情報を取得し、その情報をもとに色々設定すべきなのですが、今回はべた要素を表示させています。
概ねコードを見るとイメージは湧くかと思いますので、頑張った部分だけ言及します。
具体的には以下の部分です。

```tsx
[...Array.from(editorState._nodeMap.values())].forEach((val) => {
  /** MentionNodeでない場合は何もしない */
  if (!$isMentionNode(val)) {
    return;
  }
  /** Nodeの種類を一意に識別するKeyからHTML要素を取得 */
  const mentionElm = editor.getElementByKey(val.getKey());
  if (!mentionElm) {
    return;
  }
  /** 省略 */
  /**
   * MentionNodeを示すHTML要素に対して各種イベントハンドラーを登録する
   */
  mentionElm.addEventListener("click", (e) => {
    showInfo(e.target);
  });
  mentionElm.addEventListener("mouseenter", (e) => {
    showInfo(e.target);
  });
  mentionElm.addEventListener("mouseleave", (e) => {
    if (e.relatedTarget === hoverInfoRef.current) {
      return;
    }
    setMenuStyles(() => null);
  });
});
```

Lexical はエディターの内容を EditorState というデータで持っており、その中にあるメンション要素に対してそれぞれイベントハンドラーを登録しています。
Lexical はエディター全体に対してイベントハンドラーを設定した場合、エディタ内での登録したイベント全てに反応してしまいます。
なので、個別の要素に対してハンドラーを登録することは簡単にはできません。
さらにカスタムノードは静的な定義にとどまるので、動的にイベントハンドラーの設定が難しいです。
そこで、最新の EditorState から MentionNode を取り出すようにして、その Node から HTML 要素へ変換し、イベントハンドラーを登録しています。
これによって、メンション要素のみホバーなどした際、ホバー要素が表示されるようにしました。
力技感もありますが、指定の要素に対してのみイベントハンドラーを登録できる仕組みを導入できたのは個人的に大きな収穫です。

## メンション要素表示に検索要素を表示

コード全体

```tsx
import { getUsers } from "@/app/actions";
import { useLexicalComposerContext } from "@lexical/react/LexicalComposerContext";
import { $createTextNode, $getSelection, $isRangeSelection } from "lexical";
import { CSSProperties, useEffect, useRef, useState } from "react";
import { createPortal } from "react-dom";
import { getSelectedNode } from "../../utils/getSelectedNode";
import { $isMentionNode } from "./node";
/**
 * 文字列検索結果を表示する要素
 */
export default function SelectsFloating({
  anchorElm,
  mentionElm,
  searchText,
}: {
  anchorElm: HTMLElement;
  mentionElm: HTMLElement;
  searchText: string;
}) {
  /** 検索結果要素を設定するREf */
  const mentionMenuRef = useRef<HTMLUListElement | null>(null);
  const [selectTexts, setSelectTexts] = useState<string[]>([]);
  const [editor] = useLexicalComposerContext();
  const [isLoading, setIsLoading] = useState(false);
  /** 検索結果要素のスタイルを定義する関数 */
  const getMentionMenuStyles = (): CSSProperties | null => {
    if (!anchorElm) {
      return null;
    }
    /** メンション要素のブラウザからの高さにエディターのブラウザからの高さを引く。
     * これによって、対象のメンション要素の近くに検索結果要素を出現させることができる
     */
    const top =
      mentionElm.getBoundingClientRect().top -
      anchorElm.getBoundingClientRect().top;
    /**
     * 高さとにているが、メンション要素の幅文ずらすことでメンション要素にかぶらず表示ができる
     */
    const left =
      mentionElm.getBoundingClientRect().left +
      mentionElm.getBoundingClientRect().width +
      20 -
      anchorElm.getBoundingClientRect().left;
    return {
      top: `${top}px`,
      left: `${left}px`,
      position: "absolute",
      zIndex: 1,
      borderRadius: "8px",
      backgroundColor: "#fff",
      border: "2px solid #efefef",
      width: "200px",
      height: "fit-content",
      maxHeight: "200px",
      padding: "0.75rem 0",
      overflowY: "auto",
    };
  };
  const [menuStyles, setMenuStyles] = useState<CSSProperties | null>(null);
  useEffect(() => {
    setIsLoading(() => true);
    /** 検索関数実行 */
    getUsers(searchText)
      .then((res) => {
        /** 戻り値を検索結果要素に表示するStateに追加する */
        setSelectTexts(() => res);
        setMenuStyles(() => getMentionMenuStyles());
      })
      .finally(() => {
        setIsLoading(() => false);
      });
  }, [searchText]);
  /** 検索結果をクリックした時の挙動 */
  const handleClick = (text: string) => {
    /** エディター内の状態を書き換える
     * Lexicalの操作に使用できる、ユーティリティ関数は先頭に$が付与されている
     * そして、上記ユーティリティ関数はeditor.update か editor.read内のコールバック関数内でしが使用できない。
     */
    editor.update(() => {
      /** 現在選択しているテキストの情報やNodeの情報を持つSelectionオブジェクトを取得 */
      const selection = $getSelection();
      /** selectionオブジェクトが存在しない = 特にエディタで入力などの操作をしていない　場合は何もしない */
      if (!$isRangeSelection(selection)) {
        return null;
      }
      /** 操作対象となっているNodeを取得する */
      const node = getSelectedNode(selection);
      /** MentionNodeはElementNodeを継承しているため、テキスト部分はTextNodeである
       * そのため、MentionNode部分を取得するために、対象Nodeの親Nodeを取得するようにしている
       */
      const parent = node.getParent();
      /** メンション要素内でクリックしている場合のみ、内部のテキストを入れ替える */
      if ($isMentionNode(parent)) {
        node.replace($createTextNode(`@${text}`));
        parent.setDesidedText(`@${text}`);
        parent.insertAfter($createTextNode(" "));
        parent.selectNext();
        setMenuStyles(() => null);
      }
      /** 登録したので、検索結果要素をクリアする */
      if (mentionMenuRef?.current) {
        mentionMenuRef.current = null;
      }
    });
  };
  return (
    <>
      {!!menuStyles
        ? createPortal(
            <div style={menuStyles}>
              {isLoading ? (
                <div
                  className="flex justify-center h-full"
                  aria-label="読み込み中"
                >
                  <div className="animate-spin h-10 w-10 border-4 border-blue-500 rounded-full border-t-transparent"></div>
                </div>
              ) : (
                <ul ref={mentionMenuRef}>
                  {selectTexts.map((text) => (
                    <li
                      key={text}
                      className="hover:bg-blue-300  cursor-pointer"
                      onClick={() => {
                        handleClick(text);
                      }}
                    >
                      {text}
                    </li>
                  ))}
                </ul>
              )}
            </div>,
            anchorElm
          )
        : null}
    </>
  );
}
```

メンション機能が設定されたときに、文字列検索ができる要素を表示するようにする処理です。
具体的には以下の挙動部分です。
![mention-search-demo.gif](/images/development-lexical-editor-2024-07-01/mention-search-demo.gif)
コード自体は結構ホバー要素の時と似ています。
違う部分は以下の検索結果を選択した時にテキストを書き換える部分と

```tsx
if ($isMentionNode(parent)) {
  node.replace($createTextNode(`@${text}`));
  parent.setDesidedText(`@${text}`);
  parent.insertAfter($createTextNode(" "));
  parent.selectNext();
  setMenuStyles(() => null);
}
```

メンション要素内のテキストをもとに検索処理を実行する部分だけかなと思います。

```tsx
useEffect(() => {
  setIsLoading(() => true);
  /** 検索関数実行 */
  getUsers(searchText)
    .then((res) => {
      /** 戻り値を検索結果要素に表示するStateに追加する */
      setSelectTexts(() => res);
      setMenuStyles(() => getMentionMenuStyles());
    })
    .finally(() => {
      setIsLoading(() => false);
    });
}, [searchText]);
```

# おわりに

今回は Lexical の開発、特にメンション機能について見ていきました。
こうやって書いてみると案外やっていることは少ないですね。
ただ、そこに至るまで色々紆余曲折あったので、何とか書ききれて嬉しいです。
ここまで読んでいただきありがとうございました。
