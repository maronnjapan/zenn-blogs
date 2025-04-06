---
title: "VSCodeの安定版でGitHub Copilot Agentが使えるようになったのでFigmaのコンポーネントを実装させてみた"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MCP","Figma","GitHub"]
published: true
---
## はじめに
VSCodeのInsiders版しか使用できなかったGitHub Copilot Agenetが、先日VSCodeの安定版 v1.99でも使用できるようになりました👏
https://code.visualstudio.com/updates/v1_99
さらにMCPサーバーの設定も適用できるようになったようなので、早速試してみました。
とはいえ、何か実装があると分かりやすいかなと思い、FigmaのMCPサーバーを設定・起動し、GitHub Copilot Agent経由でアクセスできるようにしました。
そして、Figma内のコンポーネントを実装してもらいました。
今回はそれら流れを見ていこうと思います。
なお、試した環境はWindows11で、コードの実装はWSL上のプロジェクトにしてもらっています。
また、以下はできている前提でお話しますのでご注意ください。
1. WSL上のプロジェクトをVSCodeで編集できる。
2. GitHub Copilotの契約&VSCodeで使用するための設定
3. Figmaのアカウント作成
4. WindowsとWSLどちらの環境でもNodeが使用できる。

1,2は設定しないとそもそも使用がままならないので設定すると思います。
ですが、4については設定しなくてもそれっぽい所まで行けてしまうのですが、必ず設定してください。
私のPCではWindowsとWSL両方にNodeがないと、FigmaのMCPサーバーを上手く動かすことができなかったです。
なので、前提の4つは**必ず**設定してください。
## この記事の設定を行うためのダイジェスト
以下の記事でVSCodeをアップデートする方法をチェックし
[https://lifedaa.com/it/visual-studio-code/](https://lifedaa.com/it/visual-studio-code/)
以下の記事でFigmaのMCPサーバーを設定ファイルに書く方法を知り
[https://zenn.dev/takna/articles/mcp-server-tutorial-06-figma](https://zenn.dev/takna/articles/mcp-server-tutorial-06-figma)
以下の記事で具体的にどのファイルのどこに書くのかを知りました
[https://qiita.com/youtoy/items/adfeedeedf1309f194ce](https://qiita.com/youtoy/items/adfeedeedf1309f194ce)
この記事は上記3つの記事によって成り立っています。
あくまで、これらを自分の言葉でまとめたに過ぎませんので、サクッと表題の内容を構築したい場合は、各記事を参照した方が早いと思います。
上記は認識した上でこれからの内容を読んでいただけますと幸いです。
## VSCodeのアップデートとAgentモードの確認
まずはGitHub Copilot Agentを使用するために、VSCodeのバージョンアップをします。
Windowsの場合は以下のようにHelpメニューを開き、赤線部分を確認します。
![スクリーンショット 2025-04-05 232112.png](/images/github-coipilot-agent-and-figma-mcp/2025-04-96_11h11_11.png)
ここが「Restart To Update」みたいな文言であれば、クリックします。
画面のように「Check for Update」であれば多分大丈夫です。
VSCodeをアップデートしたら、GitHub Copilotのチャットを開きます。
![2025-04-05_23h22_24.png](/images/github-coipilot-agent-and-figma-mcp/2025-04-05_23h22_24.png)
すると、選択モードでAgentが追加されていれば使用できるようになっています。
なお、私の場合アップデートしたら直ぐにAgentモードは表示されておらず、何回かVSCodeをリロードしたら出てきたので、少し待ってみるのが良いかもしれません。
### 参考資料
https://lifedaa.com/it/visual-studio-code/
## FigmaのMCPサーバーアクセス準備
VSCodeでGitHub Copilot Agenetを使えるようにしたので、次にFigma内のコンポーネントへアクセスできるようにします。
そのためには、トークンを取得する必要があります。
トークンを取得するために、まず[Figma](https://figma.com/)にアクセスし左上にあるアイコンをクリックしたら、「設定」をクリックします。
![2025-04-06_12h42_51.png](/images/github-coipilot-agent-and-figma-mcp/2025-04-06_12h42_51.png)
設定をクリックした時に表示される画面で「セキュリティ」タブをクリックします。
すると個人トークンを作成できるので、任意の名前・有効期限で作成します。
スコープに関しては適切か分かりませんが、以下のようにしたら今回の作業は行うことができました。
![2025-04-06_12h47_39.png](/images/github-coipilot-agent-and-figma-mcp/2025-04-06_12h47_39.png)
「トークンを生成」をクリックしたら、表示されたトークンをどこかに保持しておきます。
### 参考資料
[https://zenn.dev/takna/articles/mcp-server-tutorial-06-figma](https://zenn.dev/takna/articles/mcp-server-tutorial-06-figma)
## FigmaのMCPサーバーの設定をVSCodeに適用する
VSCodeとFigmaそれぞれの設定はできたので、それらを繋げる設定をします。
なお、改めてですが私はWindows PCを使用しており、開発はWSL上で行っています。
そして、Windows とWSL両方でNode.jsを使用できる環境となっています。
上記前提を満たしている場合、動くことは確認していますが、それ以外の環境で動作確認はしていないので、ご了承ください。
まず、MCP上の設定ファイルを開きます。
VSCodeの左下にある歯車(⚙)をクリックして、Settingsを開きます。(Ctrl+,でも可)
開いた後、フォームにMCPと入力して一番上に出てくる結果内にある「Edit in settings.json」をクリックします。
![2025-04-06_12h52_59.png](/images/github-coipilot-agent-and-figma-mcp/2025-04-06_12h52_59.png)
すると、JSONファイルがあるのでその中でmcpプロパティを探します。
mcpプロパティ内にserversプロパティがあるので、その中に以下を追加します。
```json
"figma-developer-mcp": {
  "command": "npx",
  "args": [
    "-y",
    "figma-developer-mcp",
    "--figma-api-key=Figmaで取得したトークン",
    "--stdio"
  ]
 }
```
mcpプロパティが以下のような形になっていればOKです。
```tsx
  "mcp": {
    "inputs": [],
    "servers": {
      "figma-developer-mcp": {
        "command": "npx",
        "args": [
          "-y",
          "figma-developer-mcp",
          "--figma-api-key=Figmaで取得したトークン",
          "--stdio"
        ]
      }
    }
  
```
後は、settings.jsonの記載したMCP設定上に出てくる「Start」をクリックして実行できれば設定は完了です。
![2025-04-06_12h59_10.png](/images/github-coipilot-agent-and-figma-mcp/2025-04-06_12h59_10.png)
なお、私の場合、この「Start」する時に`spawn npx ENOENT`というエラーが発生して一向に起動できませんでした。
その要因としてWindows側にNode.jsがないことが要因だったため、何度もくどく各環境にNode.jsが必要だと記載しております。
以上で設定は完了なので、実際に操作してみます。
### 参考資料
https://qiita.com/youtoy/items/adfeedeedf1309f194ce
[https://zenn.dev/takna/articles/mcp-server-tutorial-06-figma](https://zenn.dev/takna/articles/mcp-server-tutorial-06-figma)
## 実際に開発してもらった
ここからは実際やってみた感想となります。
前提として、Viteを使用してReact環境を構築しており、GitHub Copilot Agentのモデルは「Claude 3.5 Sonnet」です。
また、参照するコンポーネントですが[こちらの記事](https://note.com/moqup_hei/n/n9016783dd2e2)にあったGmailのテンプレートを使用し、サイドメニューを実装してもらうようにしています。
![2025-04-06_13h31_26.png](/images/github-coipilot-agent-and-figma-mcp/2025-04-06_13h31_26.png)
まず以下のプロンプトをGitHub Copilot Agenetに指示します。(URLはダミーです)
```tsx
https://www.figma.com/design/aaaaaaa/gmail-design?node-id=1111&t=bbbbbbbbb
上記をFigmaのMCPサーバーからアクセスして、内部にあるサイドメニューコンポーネントを参照してそれをReactを用いて再現してください。
別途ライブラリなどが必要であればインストールしてください。
```
すると、src配下にcomponents/SideMenu.tsxを作成し、それをsrc/App.tsxで呼び出すように実装してくれました。
それを画面で表示すると、以下の表示がされました。
![2025-04-06_01h16_17.png](/images/github-coipilot-agent-and-figma-mcp/2025-04-06_01h16_17.png)
雰囲気はサイドメニューぽいですが、赤枠がはみ出しているのと数字のバランスが悪いです。
なので、次のプロンプトを渡しました。
```tsx
今作って貰ったものは悪くないですが、赤色部分がサイドメニューの大枠からはみ出しているのと数字部分が大きくバランスが悪くなっております。
そのため、今指摘した部分を修正して
```
すると、以下のような表示となる修正をしてきました。
![2025-04-06_01h19_31.png](/images/github-coipilot-agent-and-figma-mcp/2025-04-06_01h19_31.png)
大分近づいて来ました
ただ、数字は右上に表示して欲しいので、以下のプロンプトを渡し修正してもらいました。
```tsx
大分いい感じですが、数字はアイコンの右上に配置して欲しいです。
それに伴って大きさも二回り小さくして欲しいです。
```
すると、以下のようになりました。
![2025-04-06_01h21_30.png](/images/github-coipilot-agent-and-figma-mcp/2025-04-06_01h21_30.png)
デザインはほぼ完璧ですね。
ただ、実装を見るとpaddingやheithやwidthなどの値がpx固定値だったので、以下のプロンプトで内部の実装を調整してもらいました。
```tsx
デザイン自体は完璧です。
ただ、余白や幅・高さなどが現状固定値となっています。
これらスタイルをレスポンシブ対応できるような形に修正してください。
```
見た目はそのままで、内部のスタイルをpx固定にしない形で出力しれくれました。
（プロンプトがフワッとしているので、画面幅を変更しても大きなスタイルの崩れはなくとも、スマホ画面とかでも使用できるものにはなりませんでした)
最終的に出力したコードは以下の通りです。
```tsx
/** コンポーネントの呼び出し実装 */
import { Box, CssBaseline } from '@mui/material'
import SideMenu from './components/SideMenu'
import './App.css'
function App() {
  return (
    <Box sx={{ display: 'flex' }}>
      <CssBaseline />
      <SideMenu />
    </Box>
  )
}
export default App
```
本題のコンポーネントは以下の通りです。
```tsx
import React from 'react';
import { styled } from '@mui/material/styles';
import {
    List,
    ListItem,
    ListItemIcon,
    ListItemText,
    Badge,
    Paper,
} from '@mui/material';
import {
    Inbox as InboxIcon,
    Star as StarIcon,
    Snooze as SnoozeIcon,
    Send as SendIcon,
    Description as DraftIcon,
    Report as SpamIcon,
    Delete as TrashIcon,
    Label as CategoryIcon,
} from '@mui/icons-material';
const MenuContainer = styled(Paper)(({ theme }) => ({
    width: 'min(280px, 25vw)',
    height: '100vh',
    backgroundColor: '#FFFFFF',
    borderRadius: '0 clamp(1rem, 3vw, 2.94rem) clamp(1rem, 3vw, 2.94rem) 0',
    boxShadow: '0.76rem 0.76rem 3.8rem rgba(0, 0, 0, 0.05), 0 0.21rem 0.63rem rgba(60, 64, 67, 0.15)',
    overflow: 'hidden',
}));
const StyledList = styled(List)({
    padding: 'clamp(0.8rem, 2vh, 1.25rem) 0',
});
const MenuItem = styled(ListItem)<{ active?: boolean }>(({ active }) => ({
    padding: 'clamp(0.5rem, 1.5vh, 0.73rem) clamp(0.8rem, 2vw, 1.47rem)',
    minHeight: 'clamp(2.5rem, 6vh, 3rem)',
    width: 'auto',
    borderRadius: '0 clamp(1.5rem, 4vw, 2.94rem) clamp(1.5rem, 4vw, 2.94rem) 0',
    backgroundColor: active ? '#F9E9E7' : 'transparent',
    '&:hover': {
        backgroundColor: active ? '#F9E9E7' : '#F1F3F4',
        cursor: 'pointer',
    },
    '& .MuiListItemIcon-root': {
        color: active ? '#C84031' : 'rgba(0, 0, 0, 0.54)',
        minWidth: 'clamp(1.5rem, 4vw, 2rem)',
        marginRight: 'clamp(0.5rem, 1vw, 0.75rem)',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        '& .MuiSvgIcon-root': {
            fontSize: 'clamp(1.125rem, 3vw, 1.25rem)',
        }
    },
    '& .MuiListItemText-primary': {
        fontFamily: 'Roboto',
        fontWeight: active ? 700 : 400,
        fontSize: 'clamp(0.875rem, 2.5vw, 1rem)',
        color: active ? '#C84031' : '#202124',
    },
}));
const StyledBadge = styled(Badge)(({ active }: { active?: boolean }) => ({
    '& .MuiBadge-badge': {
        right: '-0.25rem',
        top: '-0.125rem',
        transform: 'none',
        backgroundColor: active ? '#C84031' : '#767676',
        color: '#FFFFFF',
        fontSize: 'clamp(0.625rem, 1.5vw, 0.75rem)',
        fontWeight: active ? 700 : 400,
        height: 'clamp(0.875rem, 2vw, 1rem)',
        minWidth: 'clamp(0.875rem, 2vw, 1rem)',
        padding: '0 0.188rem',
        lineHeight: 'clamp(0.875rem, 2vw, 1rem)',
        borderRadius: 'clamp(0.438rem, 1vw, 0.5rem)',
    },
}));
interface SideMenuItem {
    icon: React.ReactElement;
    text: string;
    count?: number;
    active?: boolean;
}
const menuItems: SideMenuItem[] = [
    { icon: <InboxIcon />, text: 'Inbox', count: 3, active: true },
    { icon: <StarIcon />, text: 'Starred' },
    { icon: <SnoozeIcon />, text: 'Snoozed' },
    { icon: <SendIcon />, text: 'Sent' },
    { icon: <DraftIcon />, text: 'Drafts', count: 1 },
    { icon: <SpamIcon />, text: 'Spam', count: 3 },
    { icon: <TrashIcon />, text: 'Trash' },
    { icon: <CategoryIcon />, text: 'Categories' },
];
const SideMenu: React.FC = () => {
    return (
        <MenuContainer elevation={0}>
            <StyledList>
                {menuItems.map((item) => (
                    <MenuItem key={item.text} active={item.active}>
                        <ListItemIcon>
                            {item.count ? (
                                <StyledBadge
                                    badgeContent={item.count}
                                    active={item.active}
                                >
                                    {item.icon}
                                </StyledBadge>
                            ) : (
                                item.icon
                            )}
                        </ListItemIcon>
                        <ListItemText primary={item.text} />
                    </MenuItem>
                ))}
            </StyledList>
        </MenuContainer>
    );
};
export default SideMenu;
```
なお、都度許可してあげる必要がありますが、ライブラリについても必要なものを選定してインストールしてくれますので、ライブラリの選定も行ってくれました。
中々いい感じに出力してくれました。
従量課金ではない&月額料金も生成AIの中で安いということから、気軽に試してみるものとしては良いと思いました。
## 個人的に気になった点
先程[こちらの記事](https://note.com/moqup_hei/n/n9016783dd2e2)にあるGmailデザインテンプレート内のサイドメニューの一部を作成してもらいました。
出来上がったものはそれぽいものができたのですが、個人的に感じた使うにあたっての課題を記載します。
### Figmaのデータ読み込みが無限ループする
サイドメニューの実装をしてもらうために、FigmaのMCPサーバーからコンポーネントを読み込んでもらおうとしました。
ただ、なぜかデータの読み込みをGitHub Copilot Agentが無限に行おうとし、エラーになってしまいました。
結構そのエラーが頻発し、参照するURLを色々と変えたら上手く動いてくれました。
エラーの原因は分からないのですが、上手くFigmaを読み込ませるのが結構煩わしかったです。
この辺は、自身の知識不足による部分もあるかと思いますので、こうやったら安定して読み込んでくれるなどありましたらコメントで教えてほしいです。
### 一々作業に許可仕上げる煩わしさ
これはセキュリティと表裏一体ではありますが、実行許可を都度尋ねてくるのは少し煩わしいと感じます。
コードの実装については、一回「Always Allow」とすればその後、実行許可を尋ねてはきません。
ただ、ライブラリのインストールは「Always Allow」みたいな設定がなく、都度許可を与える必要があります。
結構色々な方が調査していますが、一回許可をして後は勝手にやってくれという設定をGitHub Copilot Agentでやるのは無理みたいです。
勝手に破壊的な変更や、危険なライブラリをインストールしないことは確かにメリットですが、一回ドカッと作ってもらいできたものをレビューするというのは難しい気がします。
こういった大きめのタスクを渡して、一旦放置してできたものをレビューする流れができないのは、少し煩わしさを感じます。
その点Clineとかは傍若無人に実装してくれるので、上記用途ではClineの方が良さそうです。
以上のような課題は感じつつも月$10の定額料金でここまでできるのは、エージェントの進化というのを改めて感じます。
## おわりに
今回はVSCodeの安定版でも使用できるようになったGitHub Copilot AgentとFigmaを繋いで、コンポーネント実装までをしました。
正直すでに書いていただいた記事たちの焼き増しではありますが、各記事の中継点として活用いただければと思います。
ここまで読んでいただきありがとうございました。