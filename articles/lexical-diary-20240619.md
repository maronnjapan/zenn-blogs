---
title: "Lexical開発記録：リンククリックイベントの設定"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Lexical", "React"]
published: true
---

## はじめに

現在エディタを作成しており、その開発記録を記載しています。
気になる機能であれば、見て頂ければ幸いです。

## エディタ内のリンククリックで URL 遷移できるようにした

デモ
![blog-demo-move-link.gif](/images/lexical-diary-20240619/blog-demo-move-link.gif)
エディタ内のリンクはこれまでクリックしても、何も起きないようになっていました。
ですが、今回の対応でリンクをクリックした場合、対象の URL へ遷移するようにしました。

### 実装コード

```tsx
import type { LinkNode } from "@lexical/link";
import type { LexicalEditor } from "lexical";
import { $isLinkNode } from "@lexical/link";
import { useLexicalComposerContext } from "@lexical/react/LexicalComposerContext";
import {
  $getNearestNodeFromDOMNode,
  $getSelection,
  $isRangeSelection,
} from "lexical";
import { useEffect } from "react";
type LinkFilter = (event: MouseEvent, linkNode: LinkNode) => boolean;
export default function ClickableLinkPlugin({
  filter,
  newTab = true,
}: {
  filter?: LinkFilter;
  newTab?: boolean;
}): JSX.Element | null {
  const [editor] = useLexicalComposerContext();
  useEffect(() => {
    const onClick = (event: MouseEvent | PointerEvent) => {
      // クリックしたリンクの要素取得
      const linkDomNode = getLinkDomNode(event, editor);
      if (linkDomNode === null) {
        return;
      }
      // 遷移したいURLを取得
      const href = linkDomNode.getAttribute("href");
      if (!href) {
        return;
      }
      editor.update(() => {
        const selection = $getSelection();
        /**
         * selection.isCollapsed()がtrueの時は範囲選択で異なるNodeを選択している時
         * すなわち遷移したいリンク以外も選択している
         * その時は遷移したいLinkが分からなくなるので、何もしない
         */
        if ($isRangeSelection(selection) && !selection.isCollapsed()) {
          return;
        }
        // イベント経由で取得したDOMが属しているLexicalのNodeを取得する。
        // DOM自体がLexicalのNodeとして登録されたものから生成されていれば、そのNodeを返す
        // 上記の状況でない場合は、親のNodeを探していきLexicalのNodeを返す
        // 以下ライブラリのソースコード
        // https://github.com/facebook/lexical/blob/28b3f901445d730748361e008033d876991379d3/packages/lexical/src/LexicalUtils.ts#L440
        const linkNode = $getNearestNodeFromDOMNode(linkDomNode);
        if (!$isLinkNode(linkNode)) {
          return;
        }
        const isClickable =
          filter !== undefined ? filter(event, linkNode) : true;
        if (!isClickable) {
          return;
        }
        try {
          /**
           * auxclickはマウスの中央ボタンやサイドボタンをクリックした時のType
           * event.button === 1はマウスの中央ボタンをクリックしかを見ている
           */
          const isMiddle = event.type === "auxclick" && event.button === 1;
          window.open(
            href,
            newTab || event.metaKey || event.ctrlKey || isMiddle
              ? "_blank"
              : "_self",
            "noreferrer"
          );
          event.preventDefault();
        } catch {}
      });
    };
    return editor.registerRootListener(
      (
        rootElement: null | HTMLElement,
        prevRootElement: null | HTMLElement
      ) => {
        /**
         * 以下がprevRootElementにデータが格納されるケースです。
         * ①エディタの内容が変更される前に、すでにルートノードにHTMLコンテンツが存在していた場合。
         * 例えば、エディタを初期化する際に初期値としてHTMLコンテンツを設定していた場合などです。
         * ②registerRootListenerが複数回呼び出された場合。
         * registerRootListenerが呼び出されるたびに、前回のルートノードのHTMLがprevRootHtmlに渡されます。
         * これにより、前回のルートノードの状態と現在のルートノードの状態を比較することができます。
         * ③エディタの内容が動的に変更された場合。
         * ユーザーの操作やプログラムによってエディタの内容が変更された場合、prevRootHtmlにはその変更前のHTMLが存在することになります。
         */
        if (prevRootElement !== null) {
          prevRootElement.removeEventListener("click", onClick);
          prevRootElement.removeEventListener("auxclick", onClick);
        }
        // イベントの登録をしている。
        // これをしないとエディタ内でクリックしても作成した関数が実行されない
        if (rootElement !== null) {
          rootElement.addEventListener("click", onClick);
          rootElement.addEventListener("auxclick", onClick);
        }
      }
    );
  }, [editor, filter, newTab]);
  return null;
}
// イベントの対象になっている要素がaタグかを判定する
function isLinkDomNode(domNode: Node): boolean {
  return domNode.nodeName.toLowerCase() === "a";
}
// FIXME: アサーション祭りなのはどうにかしたい
function getLinkDomNode(
  event: MouseEvent | PointerEvent,
  editor: LexicalEditor
): HTMLAnchorElement | null {
  // 最新のEditor Statesを取得し、その内容をもとにNodeを取得する
  // readメソッドは型定義がread<V>(callbackFn: () => V): V;なので、任意の方を戻り値に設定できる
  // 今回はHTMLAnchorElementを戻り値に設定している
  // コールバック関数内で、対象リンクのHTML要素を取得して、それを返している
  return editor.getEditorState().read(() => {
    const domNode = event.target as Node;
    // リンクに対して遷移しようとした時はイベント内のtarget要素をそのまま返す
    if (isLinkDomNode(domNode)) {
      return domNode as HTMLAnchorElement;
    }
    // リンクNode内で文字色付与など加工されていた場合でもリンク遷移ができるように
    if (domNode.parentNode && isLinkDomNode(domNode.parentNode)) {
      return domNode.parentNode as HTMLAnchorElement;
    }
    return null;
  });
}
```

やっているの大きく分けて以下の部分です。
① エディタをクリックした時の関数を定義する。
②① の関数をエディタに登録する。
もう少し見ていきます。
なお、こちらのコードですが、[こちらの記事](https://zenn.dev/mktu/articles/6ff46411458823#%E3%81%82%E3%82%8C%E3%80%81%E3%82%AF%E3%83%AA%E3%83%83%E3%82%AF%E3%81%97%E3%81%A6%E3%82%82%E3%83%AA%E3%83%B3%E3%82%AF%E3%81%AB%E3%82%B8%E3%83%A3%E3%83%B3%E3%83%97%E3%81%97%E3%81%AA%E3%81%84)であった Lexical のプレイグラウンドをもとに実装しています。
なのですが、もう少し調べていると上記のコードは少し古く、今は lexical-react の LexicalClickablePlugin.tsx として実装されています。
そのため、今は React で Lexical を使用する場合この機能を一から実装する必要はありません。
今回はもったいないので、そのまま使用していますが…。

### クリックイベントハンドラーを登録している部分

まずは Lexical を使用したエディタにハンドラーを登録する部分を確認します。

```tsx
return editor.registerRootListener(
  (rootElement: null | HTMLElement, prevRootElement: null | HTMLElement) => {
    /**
     * 以下がprevRootElementにデータが格納されるケースです。
     * ①エディタの内容が変更される前に、すでにルートノードにHTMLコンテンツが存在していた場合。
     * 例えば、エディタを初期化する際に初期値としてHTMLコンテンツを設定していた場合などです。
     * ②registerRootListenerが複数回呼び出された場合。
     * registerRootListenerが呼び出されるたびに、前回のルートノードのHTMLがprevRootHtmlに渡されます。
     * これにより、前回のルートノードの状態と現在のルートノードの状態を比較することができます。
     * ③エディタの内容が動的に変更された場合。
     * ユーザーの操作やプログラムによってエディタの内容が変更された場合、prevRootHtmlにはその変更前のHTMLが存在することになります。
     */
    if (prevRootElement !== null) {
      prevRootElement.removeEventListener("click", onClick);
      prevRootElement.removeEventListener("auxclick", onClick);
    }
    // イベントの登録をしている。
    // これをしないとエディタ内でクリックしても作成した関数が実行されない
    if (rootElement !== null) {
      rootElement.addEventListener("click", onClick);
      rootElement.addEventListener("auxclick", onClick);
    }
  }
);
```

Lexical は通常 Lexical が許可したイベントしか実行されません。
なので、例えば検証ツールを開く Ctrl + Shift + i をクリックしても何も反応しないと思います。
もし、特別イベントを追加したい場合は Lexical へのイベント登録が必要です。
そこで使うのが、registerRootListener です。
registerRootListener メソッドがもつ要素にイベントを登録することで、エディタ内でのみ動くクリックイベントなどを定義することができます。
今回はクリック系のイベントを登録したいので、現在書き込もうとしているエディタ要素が存在すれば addEventListener で登録しています。
なお、prevRootElement が存在する場合は、本来設定したいイベントの実行回数以上実行される可能性が発生するので、removeEventListener でイベントを削除しています。

### クリック関数の中身

イベントハンドラーを登録する処理は見たので、次に以下の実際実行される関数の中身を見ていきます。

```tsx
const onClick = (event: MouseEvent | PointerEvent) => {
  // クリックしたリンクの要素取得
  const linkDomNode = getLinkDomNode(event, editor);
  if (linkDomNode === null) {
    return;
  }
  // 遷移したいURLを取得
  const href = linkDomNode.getAttribute("href");
  if (!href) {
    return;
  }
  editor.update(() => {
    const selection = $getSelection();
    /**
     * selection.isCollapsed()がtrueの時は範囲選択で異なるNodeを選択している時
     * すなわち遷移したいリンク以外も選択している
     * その時は遷移したいLinkが分からなくなるので、何もしない
     */
    if ($isRangeSelection(selection) && !selection.isCollapsed()) {
      return;
    }
    // イベント経由で取得したDOMが属しているLexicalのNodeを取得する。
    // DOM自体がLexicalのNodeとして登録されたものから生成されていれば、そのNodeを返す
    // 上記の状況でない場合は、親のNodeを探していきLexicalのNodeを返す
    // 以下ライブラリのソースコード
    // https://github.com/facebook/lexical/blob/28b3f901445d730748361e008033d876991379d3/packages/lexical/src/LexicalUtils.ts#L440
    const linkNode = $getNearestNodeFromDOMNode(linkDomNode);
    if (!$isLinkNode(linkNode)) {
      return;
    }
    const isClickable = filter !== undefined ? filter(event, linkNode) : true;
    if (!isClickable) {
      return;
    }
    try {
      /**
       * auxclickはマウスの中央ボタンやサイドボタンをクリックした時のType
       * event.button === 1はマウスの中央ボタンをクリックしかを見ている
       */
      const isMiddle = event.type === "auxclick" && event.button === 1;
      window.open(
        href,
        newTab || event.metaKey || event.ctrlKey || isMiddle
          ? "_blank"
          : "_self",
        "noreferrer"
      );
      event.preventDefault();
    } catch {}
  });
};
```

まずはイベントの target がリンク要素の場合はその要素を返すようにしています。
リンク要素ではない場合は null を返します。
リンク要素を取得する部分のコードも少しみていきます。

```tsx
function getLinkDomNode(
  event: MouseEvent | PointerEvent,
  editor: LexicalEditor
): HTMLAnchorElement | null {
  // 最新のEditor Statesを取得し、その内容をもとにNodeを取得する
  // readメソッドは型定義がread<V>(callbackFn: () => V): V;なので、任意の方を戻り値に設定できる
  // 今回はHTMLAnchorElementを戻り値に設定している
  // コールバック関数内で、対象リンクのHTML要素を取得して、それを返している
  return editor.getEditorState().read(() => {
    const domNode = event.target as Node;
    // リンクに対して遷移しようとした時はイベント内のtarget要素をそのまま返す
    if (isLinkDomNode(domNode)) {
      return domNode as HTMLAnchorElement;
    }
    // リンクNode内で文字色付与など加工されていた場合でもリンク遷移ができるように
    if (domNode.parentNode && isLinkDomNode(domNode.parentNode)) {
      return domNode.parentNode as HTMLAnchorElement;
    }
    return null;
  });
}
```

クリックしている要素が、LinkNode の要素かもしくは LinkNode 内の子要素であれば、LinkNode の DOM を探してそれを返すようにしています。
リンク取得する関数を実行した後は、遷移するために必要な値が残っているかをチェックします。

```tsx
if (linkDomNode === null) {
  return;
}
// 遷移したいURLを取得
const href = linkDomNode.getAttribute("href");
if (!href) {
  return;
}
```

遷移に必要な情報が無ければ、何もしません。
値が存在する場合は以下の処理を実行します。

```tsx
editor.update(() => {
  const selection = $getSelection();
  if ($isRangeSelection(selection) && !selection.isCollapsed()) {
    return;
  }
  const linkNode = $getNearestNodeFromDOMNode(linkDomNode);
  if (!$isLinkNode(linkNode)) {
    return;
  }
  const isClickable = filter !== undefined ? filter(event, linkNode) : true;
  if (!isClickable) {
    return;
  }
  try {
    const isMiddle = event.type === "auxclick" && event.button === 1;
    window.open(
      href,
      newTab || event.metaKey || event.ctrlKey || isMiddle ? "_blank" : "_self",
      "noreferrer"
    );
    event.preventDefault();
  } catch {}
});
```

遷移するための処理自体は window.open が行っており、遷移する URL はすでに取得しているので、これだけで遷移はできます。
ですが、以下の部分でクリックしているのがエディタ内の LinkNode であることを担保します。

```tsx
const selection = $getSelection();
if ($isRangeSelection(selection) && !selection.isCollapsed()) {
  return;
}
const linkNode = $getNearestNodeFromDOMNode(linkDomNode);
if (!$isLinkNode(linkNode)) {
  return;
}
```

そして、仮にリンク遷移するボタンの種類などを左ボタンに限定したいなどあれば、引数 filter にそのロジックを書きます。
今回は遷移さえ出来ればよいので、特に追加でロジックは設定しません。
これらを行い、リンク遷移して良い場合は以下の部分で、遷移をします。

```tsx
const isMiddle = event.type === "auxclick" && event.button === 1;
window.open(
  href,
  newTab || event.metaKey || event.ctrlKey || isMiddle ? "_blank" : "_self",
  "noreferrer"
);
```

新しいタブで開くかの設定などを付与しつつ、URL を開きます。
なお、`event.type === "auxclick" && event.button === 1;`部分はマウスの中央ボタンをクリックしているかを判定しています。
上記クリックでは別タブで開きたいので、分岐として用意しています。
以上をプラグインを束ねる部分で呼べば、リンクをクリックした時に対象リンクへ遷移できるようになります。

## おわりに

今回はリンクをクリックした時の遷移について実装しました。
調べていく中ですでにライブラリ側へ組み込まれており、もう実装する必要がなかったのは衝撃でした。
車輪の再発明をしてしまい、ちょっと残念でしたが、内部の処理について理解を深めることができたのでよしとします。
引き続き開発をおこなっていきます。
ここまで読んでいただきありがとうございました。
