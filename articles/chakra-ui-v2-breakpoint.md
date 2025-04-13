---
title: "Chakra UI v2でuseBreakpointValueが動かなくなる要因と解消方法について"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ChakraUI", "React","Breakpoint"]
published: true
---
## はじめに
Chakra UIのV2を使用しているのですが、レスポンシブデザインを実装する際に予期せぬ動作に遭遇しました。
この記事では、レスポンシブ対応の時に使用した`useBreakpointValue`フックが正しく機能しないケースについて、その原因を探り解決方法を解説していきます。
具体的には、以下のようなコードで発生する問題を見ていきます。
```tsx
import * as React from 'react'
import { ChakraProvider, extendTheme } from '@chakra-ui/react'
import { BreakPoint } from './components/BreakPoint'
import { BreakpointDebug } from './components/MatchQuery'
const breakpoints = {
  base: '0px',
  sm: '410px',
  md: '768px',
  lg: '1280px',
}
const theme = extendTheme({ breakpoints })
export default function App() {
  return (
    <ChakraProvider theme={theme}>
      <BreakPoint />
    </ChakraProvider>
  )
}
```
`BreakPoint`コンポーネントの実装は次のようになっています：
```tsx
import { Box, useBreakpointValue } from "@chakra-ui/react";
export function BreakPoint() {
    const breakpoint = useBreakpointValue({ md: 'none', lg: 'inline' })
    return (
        <div>
            <Box display={breakpoint}>aaa</Box>
        </div>
    );
}
```
期待していた動作は、768〜1279pxでは何も表示されず、1280px以上で「aaa」がインライン要素として表示されることでした。
しかし実際には、Boxコンポーネントに何もスタイルが設定されない状況が発生します。
この問題の原因と解決方法を順に見ていきます。
## 前提
議論を始める前に、いくつかの前提知識を確認します。
- この記事では、[extendTheme](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/components/src/extend-theme/extend-theme.ts#L86C1-L118C52)の基本的な使い方を理解していることを前提としています
- extendThemeの詳細については、[Chakra UIの公式ドキュメント](https://v2.chakra-ui.com/docs/styled-system/customize-theme)を参照してください
- `extendTheme`の重要な特性として、以下の点を理解しておく必要があります：
    - カスタムプロパティは基本的にデフォルトテーマを上書きします
    - カスタムプロパティで指定していないプロパティはデフォルトの値が設定されます

カスタムテーマの上書きはあくまで、一致するプロパティ部分のみ上書きするということは、この後の話に関わっているのでご認識のほどよろしくお願いします。
## 解消方法
先に、解決方法をお伝えします。
問題を解決するために必要なことは実は[ドキュメント](https://v2.chakra-ui.com/docs/styled-system/responsive-styles#customizing-breakpoints:~:text=Note%3A%20If%20you%27re%20using%20pixels%20as%20breakpoint%20values%20make%20sure%20to%20always%20provide%20a%20value%20for%20the%202xl%20breakpoint%2C%20which%20by%20its%20default%20pixels%20value%20is%20%221536px%22)に記載されていました。
> Note: If you're using pixels as breakpoint values make sure to always provide a value for the 2xl breakpoint, which by its default pixels value is "1536px"

ピクセル（px）を使用する場合は、sm, md, lg, xl, 2xlの**全ての**ブレイクポイント定義をextendThemeに渡す必要があります。
したがって、以下のように修正することで期待通りの動作が得られます：
```tsx
const breakpoints = {
  base: '0px',
  sm: '410px',
  md: '768px',
  lg: '1280px',
  xl: '1440px',
  '2xl': '1536px',
}
```
一方で、emを使用する場合は全てのプロパティを渡さなくても正しく動作します：
```tsx
const breakpoints = {
  base: '0em',/** 0px */
  sm: '25.625em',/** 410px */
  md: '48em',/** 768px */
  lg: '80em',/** 1280px */
}
```
以上で問題は解決しますが、「なぜpxの場合は全てのプロパティを定義する必要があり、emの場合はそうでないのか」という疑問が残ります。
これからその理由を深掘りしていきましょう。
問題の理由としては、Chakra UIが内部的にemを前提として処理しているため、一部のプロパティだけをpxで指定すると、ソートの順序に影響が出て、生成される@mediaクエリの範囲が想定と異なってしまうことがあげられます。
## useBreakpointValueからBreakpointを扱っている部分までを辿る
まずは、[`useBreakpointValue`のソースコード](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/components/src/media-query/use-breakpoint-value.ts)の処理部分を見てみます。
```tsx
export function useBreakpointValue<T = any>(
  values: Partial<Record<string, T>> | Array<T | null>,
  arg?: UseBreakpointOptions | string,
): T | undefined {
  const opts = isObject(arg) ? arg : { fallback: arg ?? "base" }
  const breakpoint = useBreakpoint(opts)
  const theme = useTheme()
  if (!breakpoint) return
  const breakpoints: string[] = Array.from(theme.__breakpoints?.keys || [])
  const obj = Array.isArray(values)
    ? Object.fromEntries<any>(
        Object.entries(arrayToObjectNotation(values, breakpoints)).map(
          ([key, value]) => [key, value],
        ),
      )
    : values
  return getClosestValue(obj, breakpoint, breakpoints)
}
```
ここでは、`useBreakpoint`関数と`useTheme`関数が呼び出されています。
`useBreakpoint`関数の方が関係ありそうですが、今回着目するのは主に[`useTheme`](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/components/src/system/use-theme.ts)の部分です。
[`useTheme`](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/components/src/system/use-theme.ts)の実装の一部は以下のようになっています。
```tsx
import { ThemeContext } from "@emotion/react"
export function useTheme<T extends object = Dict>() {
  const theme = useContext(
    ThemeContext as unknown as React.Context<T | undefined>,
  )
  if (!theme) {
    throw Error(
      "useTheme: `theme` is undefined. Seems you forgot to wrap your app in `<ChakraProvider />` or `<ThemeProvider />`",
    )
  }
  return theme as WithCSSVar<T>
}
```
この関数は、Emotionの`ThemeContext`から値を取得しているのが分かります。
この`ThemeContext`の値は[`ThemeProvider`](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/components/src/system/providers.tsx#L21C1-L30C2)コンポーネントによって提供されます。
```tsx
export function ThemeProvider(props: ThemeProviderProps): JSX.Element {
  const { cssVarsRoot, theme, children } = props
  const computedTheme = useMemo(() => toCSSVar(theme), [theme])
  return (
    <EmotionThemeProvider theme={computedTheme}>
      <CSSVars root={cssVarsRoot} />
      {children}
    </EmotionThemeProvider>
  )
}
```
見ていただきたいのは、propsで受け取った`theme`が`toCSSVar`関数に渡されて変換され、その結果が`EmotionThemeProvider`に設定されている点です。
`useTheme`関数はこの変換後の`computedTheme`を取得しています。
なお、ThemeProviderへthemeがChakraProviderからどうわたっているかは[chakra-provider.tsx](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/components/src/chakra-provider.tsx)→[create-provider.tsx](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/components/src/provider/create-provider.tsx)→[provider.tsx](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/components/src/provider/provider.tsx)→[providers.tsx](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/components/src/system/providers.tsx)と追ってもらえれば分かりますので、良ければご参照ください。
話を戻すと、`computedTheme`に使用された[`toCSSVar`関数](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/styled-system/src/create-theme-vars/to-css-var.ts)の一部を見ると、[ブレイクポイント処理に関わる部分](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/styled-system/src/create-theme-vars/to-css-var.ts#L40)がありますので、コードの一部を記載します。
```tsx
import { analyzeBreakpoints } from "@chakra-ui/utils"
export function toCSSVar<T extends Record<string, any>>(rawTheme: T) {
  Object.assign(theme, {
    __breakpoints: analyzeBreakpoints(theme.breakpoints),
  })
  return theme as WithCSSVar<T>
}
```
ここで[`analyzeBreakpoints`関数](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/styled-system/src/create-theme-vars/to-css-var.ts#L40)が呼び出され、その結果が`__breakpoints`プロパティに設定されています。
この関数が今回の問題と関わりが深い部分ですので、後の章で取り扱います。
## （余談）useBreakpointValueが一致するBreakpointを取得する処理について
本題から少し脱線しますが、ブレイクポイントの判定処理についても簡単に触れます。
Breakpointを判定し、一致する値を取得する処理は先程スルーした[useBreakpoint関数](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/components/src/media-query/use-breakpoint.ts)がになっています。
なので、実装を見ていくため、`useBreakpoint`関数の主な部分をコメント付きで抽出します。
```tsx
export function useBreakpoint(arg?: string | UseBreakpointOptions) {
  /** 先程解説したBreakpointsの値を持つthemeを取得 */
  const theme = useTheme()
	/** 
	*	Breakpoints部分の値を抽出しBreakpoint名(base,smなど)と 
	* 「@media screen and (min-width: 320px) and (max-width: 480px)」といった形で記載されている文字列を持つminMaxQuery
	* これら二つの値を抽出している
	* ※内部の処理は後述の章にて解説
	*/
  const breakpoints = theme.__breakpoints!.details.map(
    ({ minMaxQuery, breakpoint }) => ({
      breakpoint,
      query: minMaxQuery.replace("@media screen and ", ""),
    }),
  )
	/** 
	* useMediaQuery関数にqueryを渡している
	* useMediaQueryは画面幅がqueryに記載されているmin-width~max-widthの範囲内であれば
	* trueを返すhook関数である
	*/
  const values = useMediaQuery(
    breakpoints.map((bp) => bp.query),
  )
  /** useMediaQuery関数でtrueとなっている番号を取得し、一致するBreakpointの値を返す */
  const index = values.findIndex((value) => value == true)
  return breakpoints[index]?.breakpoint ?? opts.fallback
}
```
この関数は`useMediaQuery`を使用して、現在の画面幅がどのブレイクポイント範囲に該当するかを判断しています。
`useMediaQuery`関数内では[`window.matchMedia`](https://developer.mozilla.org/ja/docs/Web/API/Window/matchMedia)を使用して、[メディアクエリと現在の画面幅が一致するかをチェック](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/components/src/media-query/use-media-query.ts#L41C4-L46C6)しています。
```tsx
setValue(
  queries.map((query) => ({
    media: query,
    matches: win.matchMedia(query).matches,
  }))
);
```
`window.matchMedia`の`matches`プロパティは、画面幅が指定された範囲内にあれば`true`を返します。
この仕組みにより、現在の画面幅に該当するブレイクポイントを特定できるようになっています。
## analyzeBreakpoints関数の分析
ここからは`analyzeBreakpoints`関数について詳しく見ていきます。
先程の余談で、useBreakpointValue周りのBreakpointの判定は__breakpointsプロパティ内のdetails部分の値を使うことが分かったので、当該部分を中心に[analyzeBreakpoints関数](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/utils/src/breakpoint.ts#L56C1-L135C2)のコードを抽出します。
```tsx
export function analyzeBreakpoints(breakpoints: Record<string, any>) {
  if (!breakpoints) return null
  breakpoints.base = breakpoints.base ?? "0px"
  const queries = Object.entries(breakpoints)
    .sort(sortByBreakpointValue)
    .map(([breakpoint, minW], index, entry) => {
      let [, maxW] = entry[index + 1] ?? []
      maxW = parseFloat(maxW) > 0 ? subtract(maxW) : undefined
      return {
        _minW: subtract(minW),
        breakpoint,
        minW,
        maxW,
        maxWQuery: toMediaQueryString(null, maxW),
        minWQuery: toMediaQueryString(minW),
        minMaxQuery: toMediaQueryString(minW, maxW),
      }
    })
  return {
    details: queries,
  }
}
```
この関数は次のことを行います。
1. 受け取った`breakpoints`オブジェクトを`Object.entries`で`[key, value]`のペアの配列に変換
2. `sortByBreakpointValue`関数でソート
3. 各ブレイクポイントに対して、`toMediaQueryString`関数でメディアクエリの文字列を生成

Object.entriesは分割するだけなので、今回関わっていそうなのはsort(sortByBreakpointValue)とmaxW取得部分とtoMediaQueryString関数ぽそうです。
ただ、`toMediaQueryString`関数は[ソースコード](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/utils/src/breakpoint.ts#L47C1-L54C2)を見てもらえば大体雰囲気はつかめるので解説は省略します。
そのため、ソートとmaxWの取得方法を見ていきます。
まずソート部分からです。
`sortByBreakpointValue`関数は以下のように定義されています。
```tsx
const sortByBreakpointValue = (a: any[], b: any[]) =>
  parseInt(a[1], 10) > parseInt(b[1], 10) ? 1 : -1
```
これは`parseInt`を使用して、ブレイクポイントの値を10進数として解釈し、小さい順にソートしています。
ここが注目すべき点です。
`parseInt`は入力値から数値部分のみを取得するため、`10em`と`15px`のような異なる単位の値が混在すると、単位を無視して数値のみでソートされます（この場合は`10`と`15`として解釈され、`15px`が大きい値とみなされます）。
また、`maxW`の取得部分も確認します。
```tsx
let [, maxW] = entry[index + 1] ?? []
maxW = parseFloat(maxW) > 0 ? subtract(maxW) : undefined
```
ここでは、ソートされた配列の次の要素を取得し、それをその区間の上限値として使用しています。
この処理は、配列がすでに正しくソートされていることを前提としています。
そのため、ソートの順序に問題があると、範囲設定も想定と異なる結果になってしまいます。
## 関数が受け取るbreakpointsとソートについての問題
ここからが問題の要点です。
`analyzeBreakpoints`関数は、[デフォルトテーマ](https://www.notion.so/Chakra-UI-V2-useBreakpointValue-1d327889392e80bfb250d2f898969fe4?pvs=21)とカスタムテーマを組み合わせたブレイクポイント設定を受け取ります。
Chakra UIのデフォルトブレイクポイント設定は[こちらの定義](https://github.com/chakra-ui/chakra-ui/blob/v2/packages/theme/src/foundations/breakpoints.ts)なっています。
```tsx
const breakpoints = {
  base: "0em",
  sm: "30em",
  md: "48em",
  lg: "62em",
  xl: "80em",
  "2xl": "96em",
}
```
そして、今回のカスタムテーマは以下のように設定しています。
```tsx
const breakpoints = {
  base: '0px',
  sm: '410px',
  md: '768px',
  lg: '1280px',
```
`extendTheme`は上書きの仕組みを使うため、最終的なブレイクポイント設定は以下のようになります。
```tsx
const breakpoints = {
  base: '0px',
  sm: '410px',
  md: '768px',
  lg: '1280px',
  xl: "80em",
  "2xl": "96em",
}
```
これを`Object.entries(breakpoints).sort(sortByBreakpointValue)`でソートすると：
```tsx
[
  ["base", "0px"],
  ["xl", "80em"],
  ["2xl", "96em"],
  ["sm", "410px"],
  ["md", "768px"],
  ["lg", "1280px"]
]
```
このような順序になります。
上記結果になることを実際に動かして試せる`analyzeBreakpoints`関数を模した[playground](https://www.typescriptlang.org/play/?ts=5.7.3#code/GYVwdgxgLglg9mABAQzMgNgTwF4FMDCAyoQGoYi4AUAbubgFyJggC2ARrgE6IA+iAzlE4wwAcwCUiAN4AoRIggJBTVogC8iAA7JO-XADF0cZFBp0AdFDiEhI0ZXHi5CpVETgYbjbXQVL12zEHc05cTXRkCCobYSDmFnEAGkQAIhSneVCoEE4kKXcwT3Rcfn5GAEIPKGSfCkZ45KrEAF8ZVplQSFgELQAPM18GFXYuXgFA0THmdHRxRkFYyb5p9GlnGGBEAYp1DRXJLJykWtwUfnHFqZAZ50UwZXyq4tKW9RQ0LDwiUjpt3AzEIdcgUiiVzjw+FBMJpcHBNiddhoUvEOJwUogAPyIAAGABIpCdmppetjEIwTm0ZB1wNB4EgWJh9Jw4CwAKJgWwlSi4DnCEr0VCYSSyTK4bLAnmc-ghXAAExAUUolEiEEY+QA2r15hMALqMQUtZLqgDWuEwNToeveQvUAD41vJ5CqTWadW8TgBuZyi8VIFVe+TNZJSZriL3tGR3ZT8OCcKAAIUw8dCyGNmjgIigZEGb2V+rAmHVOuSbHzhZ1kjUtuc2l0uAAkhzleqAIzFxAtgAMknttb0jdMbFb7a7kixLbJiAAtC2qdSunSBCA2EJIqYTtrFsL1ptKOUTgcxUdEBT5AiNMS-mOsRTbq5EAB5fT6QisgAqbynnfMnYATFSfWPKEYThE86ERVIUS4FJvUxHF8QRABqR9n1fN9mmxWDyQsUJwkiKgAHpKAAHVlRDiPMDFSIAKnEAiQGSSgEjteCpD7AwjBMJjJGQp8X3fDCnHaTpaR6KwAFk5RgZAAEUKE4TAYjsJiRE3OwrhmZIWGQXoMTUsRt3kKM3AAR3kzA3nVFIAAEWCk5ABAgUIeRSHUAMQDYthYERJDMrhMHMTQQH4AALSgUlQWUUmSbEVLAKcAHcYFlKAQsYfFL28sBxFDbEAU8pidN88zAuCsKIrAKKYsK3pEuS1L0rY-ptN6HLxDy9ygSQPyFPMAArDMwHC1IhKpETuj9D4cFwZNcFTdNM34Sg2BTNNBqgMpEAAJVwRROFlAAeBY7GSQVbUMjzd3KFa5rWxbD19FQbmcG75vW6U2GQPQ3leu6OQ+r7TgxLEUk7YkYLve44GKcwjHsB82D63aoHMSU+SW36Fv+8RzBjONKDxhMk1WrGszoRx3OMxAepgEo3gRpHoFR3laYxkn3oBeRcdjUxCcTWa3szbMKE5xBzG0zQlXVTH1q0kQAHV2xEWVcF6ZI0ZtKsHUdeRijcdUtJ0xW3g19VldVxBkLbTEsSLWD5Ba+W3nYwxjFMR2e0QTs4P4ZdV2gGr5ckRhwBV4ARDle3ASPYERR1x0AH0svl+Y-c4Nc4qDxIo-kGXM2z+PHWTgvC8QR2S8Lx25P8xgJPs6uFKUuJrnQQ3eiznOy4VhvMFruBJNlaSe6b+xk6STusvEnSe77geh-MkfM7bjvC9aHXQ06mO8lglWoGQGB0E2mmShL1p2ijaHcFhuB7FQDBpoFv6NsoOPPr0RgAHIwd6D+S-4FhP4ABYuzEl-s4FgspP4AHYABsAAOUBJd0CiE-i2X8cDv5gPkL0dAjAUgYNwCwaKzgUi-hwSkPBABOGBhDiGhgxOYXe+9D7i2QJLJhB8qwv0ntPcy9AOHoHFiIKevQe6hkcEAA)も用意しているので、実際に試してもらえると幸いです。
先程の配列の結果は、後続の処理に影響を及ぼします。
単位が混在しているため、ソートの際に使用された`parseInt`による数値解釈で`"80em"`が`"410px"`よりも先に配置されています。
このソート結果から生成されるメディアクエリとBreakpointは次のようになります。
```tsx
[
  {
    "breakpoint": "base",
    "minMaxQuery": "@media screen and (min-width: 0px) and (max-width: 79.98em)"
  },
  {
    "breakpoint": "xl",
    "minMaxQuery": "@media screen and (min-width: 80em) and (max-width: 95.98em)"
  },
  {
    "breakpoint": "2xl",
    "minMaxQuery": "@media screen and (min-width: 96em) and (max-width: 409.98px)"
  },
  {
    "breakpoint": "sm",
    "minMaxQuery": "@media screen and (min-width: 410px) and (max-width: 767.98px)"
  },
  {
    "breakpoint": "md",
    "minMaxQuery": "@media screen and (min-width: 768px) and (max-width: 1279.98px)"
  },
  {
    "breakpoint": "lg",
    "minMaxQuery": "@media screen and (min-width: 1280px)"
  }
]
```
`base`の次に`xl`が配置され、`base`のMediaクエリのMaxWidthが約80em（一般的には1280px相当）になってしまいます。
つまり、1280pxまでの画面幅では常に`base`のブレイクポイントが適用され、カスタム定義した`sm`や`md`が想定通りに機能しなくなってしまいます。
これこそが、pxを部分的にカスタムテーマとして与えると上手く動かなくなる要因です。
これは内部の動作を見ないと絶対に分からない挙動だなと感じました。
もうちょっと型定義で制限したい気持ちもありますが、上手い方法は思い浮かびませんでした。
とはいえ、要因の詳細まで分かったので一旦は満足できとします。
## （余談）useBreakpointValueの設定が適切か確認する方法
useBreakpointValueが想定通りのBreakpointの設定になっているかを確認するには、`useTheme`フックを使って以下のようなコードを実装すると便利です。
```tsx
import { Box, useBreakpointValue, useTheme } from "@chakra-ui/react";
export function BreakPoint() {
    const theme = useTheme();
    console.log("Theme breakpoints:", theme.__breakpoints);
    return (
			/** 任意のコンポーネント */
    );
}
```
これにより、ブレイクポイント関連の設定値を確認でき、問題の切り分けに役立ちます。
## おわりに
今回はChakra UI V2で提供されている`useBreakpointValue`が想定通りに動作しない問題の原因と解決方法を詳しく見てきました。
Chakra UIは多くのファイルと複雑な構造を持っているため、問題の切り分けには苦労しましたが、最終的に原因を見つけることができて満足です。
この問題の要点をまとめると：
1. Chakra UIは内部的にemを前提として処理している
2. pxとemが混在すると、ソート処理の順序に影響が出る
3. 結果として、生成されるメディアクエリの範囲が想定と異なってしまう

解決策としては、pxを使用する場合は必ず全てのブレイクポイント（sm, md, lg, xl, 2xl）を定義するか、全てemで統一することが有効です。
この記事が、同様の問題に悩む方々の助けになれば幸いです。
ここまで読んでいただき、ありがとうございました。