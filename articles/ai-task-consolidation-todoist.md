---
title: "Claude・ChatGPT・GeminiとTodoistを連携し、可能な限り一元的にタスクを管理するようにした"
emoji: "📋"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Todoist", "Claude", "ChatGPT", "Gemini", "タスク管理"]
published: true
---

## はじめに

### この記事で書きたいこと

普段、Claude、ChatGPT、Geminiを使い分けています。
それぞれ得意な場面が違うので自然と併用するようになったのですが、使い続けているうちに一つ困ったことが出てきました。
AIとの会話の中で「これ後でやろう」と思うタスクが、あちこちに散ってしまうことです。

そこで、可能な限りタスクの受け皿を一つにまとめたいと考えました。
今回は、その受け皿としてTodoistを使っている自分のやり方を書きます。

### この記事は誰向けか

- 生成AIを複数使い分けている人
- AIとの会話の中で「これ後でやるべきだな」というタスクが頻繁に発生する人
- タスク管理をきっちり運用するのが苦手な人
- タスクを手で丁寧に整理するより、できるだけAI側に寄せたい人

### この記事で伝えたい結論の先出し

先に結論を書いておきます。

- タスク管理を全部自分で頑張るより、生成AIと連携したほうが自分には合っていた
- 完全な一元化は難しいが、かなり負担は減らせる
- 特に複数AIを使う人ほど、「タスクの受け皿を一つ決める」価値は大きい

:::details 手順だけ知りたい人向け（クリックで開く）

背景や考え方は飛ばして設定手順だけ知りたい方は、以下を参照してください。
いずれもTodoistのアカウントが必要です。

#### Claude（MCPコネクタ経由）

1. Claudeの設定画面 →「コネクタ」→「カスタムコネクタを追加」
2. 任意の名前を入力し、リモートMCPサーバーURLに `https://ai.todoist.net/mcp` を入力
3. OAuthでTodoistアカウントを連携

前提：ClaudeのProプランなどMCPが利用できるプランが必要です。
詳細は[Claude連携の設定手順](#claude連携の設定手順)を参照してください。

#### ChatGPT（MCPアプリ経由）

1. ChatGPTの設定画面 →「アプリ」→「高度な設定」
2. 「開発者モード」を有効にする
3. 「アプリを作成する」から、名前とMCPサーバーのURL（`https://ai.todoist.net/mcp`）を入力
4. 認証方式はOAuthのまま、Todoistアカウントを連携

注意：開発者モードを有効にすると、メモリなどの一部機能が使えなくなります。
詳細は[ChatGPT連携の設定手順](#chatgpt連携の設定手順)を参照してください。

#### Gemini（Google Tasks + GAS経由）

GeminiはTodoistと直接連携できないため、Google Tasksを中継して同期します。

1. Google Apps Script（GAS）でプロジェクトを作成
2. GASエディタの「サービス」から `Tasks API` を追加
3. 「プロジェクトの設定」→「スクリプトプロパティ」に `TODOIST_API_TOKEN` を追加
4. 同期スクリプトをコピーして貼り付け
5. `checkSetup()` を手動実行して接続確認
6. `syncTasks()` に時間ベーストリガー（5分ごと）を設定

詳細とスクリプト全体は[Gemini連携の設定手順](#gemini連携の設定手順)を参照してください。

:::

## そもそも、自分はタスク管理が得意ではない

### タスク管理を適切に行うこと自体が難しかった

タスク管理の本やノウハウは世の中にたくさんあります。
読んでみると「なるほど」と思うのですが、自分の運用に落とし込めたことがほぼありませんでした。
解像度高く整理すること自体が続かないのです。
「正しいやり方」は分かっても、実際には回せない。
そんな状態がずっと続いていました。

### タスクを整理・維持し続ける行為そのものが面倒だった

タスクを作る、リストに入れる、見返す、適切に直す。
この一連の管理コストが重かったのだと思います。
一つひとつは小さい作業ですが、それを日常的に繰り返し続けることが、自分には合っていませんでした。

### 精度よりも、続けられることのほうが大事だった

完璧な整理より、雑でも残るほうがいい。
少し精度が下がっても、AIに投げられるほうが自分には合っていました。
きれいな運用より、摩擦が少ない運用を優先したかったのです。

## 発想を変えて「管理を頑張る」のをやめた

### タスク管理の一部を生成AIに任せる発想に変わった

自分で常に整理するのではなく、AIに雑に投げてしまうという方向に発想が変わりました。
タスクの言語化や整形もAIに任せて、必要に応じてAI側でごにょごにょしてもらえばいいのではないかと考えたのです。

### 自分にとって大事だったのは"ちゃんと管理すること"ではなく"取りこぼさないこと"

頭の中に浮かんだものをすぐ逃がせること、その場の会話から自然にタスク化できること、手で整える前提にしないこと。
振り返ると、自分にとって重要だったのは「管理の質」ではなく「取りこぼさないこと」でした。

### この方向性は、自分にはかなり相性がよかった

実際にやってみると、タスク入力の心理的負担が減りました。
自分で整理するコストも減りました。
タスク管理のハードルが下がったことで、結果的にタスクが残る量が増えたのは嬉しい誤算でした。

## 最初の起点はClaudeだった

### Claudeをよく使っている背景

自分はエンジニアで、開発時にClaudeを使うことが多いです。
仕様整理や実装方針のやり取りをClaudeとする場面がよくあります。

### 開発の会話の中で、そのままタスクを残したくなった

仕様を考えていると、後でやることが自然に出てきます。
その場では考え続けたいけれど、思いついた作業はどこかに残しておきたい。
会話の流れを切らずにタスク化したい、という気持ちがありました。

### ClaudeとTodoistの連携を試したのがスタートだった

まずはClaudeからTodoistへ流せないか調べてみました。
実際にやってみたら、自分にはかなりしっくりきました。
「タスク管理そのものを先生役に任せる」という感覚が合っていたのだと思います。

### ここで、生成AI連携によるタスク管理の手応えを感じた

思考の流れを切らずにタスク化できますし、自分で整理しきれなくても一旦外に出せます。
これなら続けられそうだと思えました。

## ただ、Claudeだけでは終わらなかった

### 普段使っている生成AIは一つではない

実際のところ、Claudeだけで完結してはいません。
ChatGPTもGeminiも日常的に使っています。

### 生成AIごとに役割が違う

#### Claude

主にプログラミングや仕様検討で使っています。
開発まわりの思考整理や、実装前後の壁打ちに向いていると感じています。

#### ChatGPT

音声会話での利用が個人的にかなり相性がいいです。
基本的な相談や、雑多な質問、リストアップや日常的な思考整理に使っています。

#### Gemini

NotebookLMとの連携があるのが大きいです。
資料作成や整理、スライドや情報整理の文脈で使うことがあります。

### AIを使い分けると、タスクも分散しやすい

Claudeで生まれたタスク、ChatGPTで生まれたタスク、Geminiで生まれたタスク。
それぞれ別の場所に溜まると管理がつらくなります。

### AIごとに別々のToDoを持つと、それはそれで破綻しやすい

使用するAIによってタスクの置き場所が変わってしまいます。
「どこに入れたっけ」が起きます。
せっかくAIで楽をしたいのに、確認先が増えてしまうのは本末転倒です。

### だから、タスクの"マスター"は一つに寄せたかった

完全一元化でなくてもよいのですが、可能な限り集約したいと考えました。
少なくとも「最終的に見る場所」は絞りたかったのです。

## その受け皿として、自分はTodoistを使っている

### 先に言うと、Todoistが絶対の正解だとは思っていない

この記事は、生成AI連携に最適なタスクツールを比較検討したものではありません。
Todoistがベストかどうかは保証できません。
あくまで自分の運用上、使い慣れていたから採用しています。

### Todoistを使っている一番大きな理由は、単純に使い慣れているから

以前から使っていて、日常の中で無理なく開けるツールでした。
新しいタスクツールに適応するより楽だったというのが正直なところです。

### それでも、自分がタスク管理に欲しい要素は一通り揃っていた

- タスクタイトル
- 説明欄
- 優先度
- ラベル
- 期限

### 自分にとっては、その5要素を扱えることが重要だった

タイトルだけでは足りません。
補足説明を持たせたいし、種類分けもしたい。
期限感も持たせたいけれど、管理画面は重すぎないほうがいい。
Todoistはそのバランスがちょうどよかったです。

### 今回の主眼は「Todoistが最強か」ではなく「使い慣れた道具をAIとつなぐこと」

タスクツール選定論をやりたいわけではありません。
自分が使っている環境で、どう連携を成立させるかに焦点を当てています。

### なので、この先は"Todoistを使う前提の実践例"として読んでほしい

他ツールでも置き換え可能な部分はあるはずです。
ただし、この記事ではTodoist前提で話を進めます。

## 今回やりたいことの全体像

### やりたいことはシンプル

Claude、ChatGPT、Geminiを使っていて発生するタスクを、可能な限りTodoistに寄せて、Todoistをタスクのマスターとして扱います。
やりたいことはこれだけです。
イメージは以下の形です。
![](/images/ai-task-consolidation-todoist/todoist-integrate.png)
ClaudeとChatGPTはさておき、Gemini部分は少々経由するものがあります。
詳細は後ほど説明しますので、いったんは画像の形での達成を目指したと理解いただけますと幸いです。

### ただし"完全な一元化"は目標ではない

現実にはAIごとに制約があります。
先の画像を見ても分かるように、連携方法もきれいに揃いません。
それでも、バラバラよりかなりマシかなとは思っています。

### 目指しているのは"厳密な統一"より"運用可能な集約"

多少ズレてもよいので、手間が減ることを優先しています。
現実的に回ることを重視する、そういうスタンスで進めています。

## ClaudeとTodoistをどう連携するか

### Claudeでやりたいこと

会話中に出たタスクをそのままTodoistへ送りたい、仕様検討や開発の流れを切らずにタスク化したい、後で見返せるように残したい。
この3つが主な目的です。

### Claudeを起点にする利点

開発文脈との相性がよく、タスクが発生する場面が明確なので、一番最初に連携の価値を感じやすいところでもあります。

### Claude連携の設定手順

ClaudeとTodoistの連携には、カスタムコネクタを使います。
Claudeの設定画面からコネクタに移動し、以下のカスタムコネクタの追加ボタンをクリックします。
![](/images/ai-task-consolidation-todoist/claude-add-custom-connect.png)
以下のようなモーダルが表示されるので、そこに任意の名前とリモートMCPサーバーURLには[https://ai.todoist.net/mcp](https://ai.todoist.net/mcp)を入力します。
![](/images/ai-task-consolidation-todoist/claude-custom-connect-modal.png)
リモートMCPサーバURLに使用しているものの詳細は以下のリポジトリ参照でお願いします。
https://github.com/Doist/todoist-ai

コネクタの追加が完了し、OAuthを使用した連携が完了すると、Claudeとの会話の中で「これをTodoistに追加して」と伝えるだけで、タスクがTodoistに登録されるようになります。
タスクのタイトル、説明、優先度、ラベル、期限といった情報も、会話の文脈からClaudeが適切に判断して設定してくれます。

前提条件としては、Todoistのアカウントがあること、ClaudeのProプランなどMCPが利用できるプランであること、が必要です。

### Claude連携で自分が便利だと感じたポイント

思いついた作業をすぐ残せること、会話から自然にタスク化できること、自分で後から転記しなくてよいこと、この3つが便利だと感じています。
特に「転記しなくてよい」というのは、想像以上に楽でした。

## ChatGPTとTodoistをどう連携するか

### ChatGPTでやりたいこと

音声会話の中で出たタスクを処理したい、日常的な相談や雑談の延長でタスクを残したい、雑多な依頼からやることを切り出したい。
ChatGPTでの利用は、こうした場面が多いです。

### ChatGPT連携の設定手順

ChatGPTもMCPサーバー経由でTodoistと接続できます。

ChatGPTの設定画面から、アプリを選択します。
画像のような「高度な設定」を選択します。
![](/images/ai-task-consolidation-todoist/chatgpt-app-advance-config.png)
選択後は以下の画面が表示されるので、開発者モードを有効にします。
![](/images/ai-task-consolidation-todoist/activate-chatgpt-dev-app-config.png)
有効後は、「アプリを作成する」が表示されるので、それを選択します。
選択後は以下モーダルが表示されるので、そこにClaudeと同様の値を入力します。
![](/images/ai-task-consolidation-todoist/create-chatgpt-custom-app.png)
なお、認証方式はOAuthのままで連携できるので、そのままにしてください。
設定が完了すれば、ChatGPTとの会話中に「これタスクに追加して」と伝えることで、Todoistにタスクが登録されます。
なお、この方式は開発者モードを有効にする必要があるので、メモリなどの一部機能が使えなくなるのでご注意ください。

### ChatGPT連携で便利な場面

口頭で出たやることをそのまま外に出せるのが一番大きいです。
思考整理からタスク生成までつながる感覚があります。
雑に投げても残しやすいので、「後で整理しよう」と思って忘れることが減りました。

## GeminiとTodoistをどう連携するか

### 先に結論：GeminiはClaudeやChatGPTと同じ感覚では連携できない

ここは最初に明示しておきます。
GeminiとTodoistの連携は、ClaudeやChatGPTほどシンプルにはいきません。
「できること」と「できないこと」を分けて読んでもらえればと思います。

### Geminiを使う理由

NotebookLMとの連携があることが大きいです。
資料作成や情報整理の文脈で便利ですし、スライドまわりで使うケースもあります。

### それでもGemini起点のタスクをTodoistに寄せたかった

普段のワークフロー上、Geminiを無視できないので、Geminiで生まれたタスクだけ別管理にしたくありませんでした。
完全ではなくても、できるだけ統合したかったのです。

### GeminiからTodoistへ直接つなぐ方法は、自分は見つけられなかった

少なくとも自分の調査範囲では、GeminiからTodoistへ直接連携する方法は見つけられませんでした。
直接Todoist APIを叩くルートが見つからなかったのです。
ClaudeやChatGPTと同じノリではいけませんでした。

### そこで、Google Tasksを中継地点として使う

GeminiはGoogle Tasksとのやり取りを前提にします。
全体の流れは以下のとおりです。

- Gemini → Google Tasks
- Google Tasks → Google Apps Script
- Google Apps Script → Todoist API

この流れでTodoist側に寄せています。

### Gemini連携の全体フロー

全体の構成は以下のようになっています。

```
Gemini
  ↓ タスク登録
Google Tasks
  ↑↓ GAS（5分ごとにポーリング）
Todoist（マスター）
```

GeminiはGoogle Workspaceと深く統合されているため、タスク管理にGoogle Tasksを使うのが自然な流れになります。
Geminiとの会話中に「これをタスクに追加して」と伝えると、Google Tasksにタスクが作成されます。
その後、Google Apps Script（GAS）で定期実行するスクリプトが、Google Tasksの内容をチェックします。
新しいタスクがあれば、Todoist APIを叩いて転送・同期します。

前提として、Todoistが正（マスター）です。
Google Tasksはあくまで入力窓口・ミラーとして扱っています。

### 設計上の主な判断

ここで、同期の仕組みを作るにあたって考えたことをいくつか書いておきます。

#### なぜポーリングか

Google Tasksにはタスクの作成・編集・削除をトリガーにする仕組みがありません。
WebhookのようなAPIが用意されていないのです。
そのため、GASの時間ベーストリガーで5分ごとに同期を走らせるアプローチをとっています。
リアルタイム性は若干犠牲になりますが、Geminiからのタスク登録用途では十分許容できると判断しました。

#### 「新規タスク」と「既存タスク」をどう区別するか

Google Tasks側のタスクに`[todoist:タスクID]`というタグをnotesフィールドの末尾に書き込むようにしています。
このタグの有無で新規か既存かを判定します。

- タグなし → 新規タスク（TodoistへPOSTし、IDを書き戻す）
- タグあり → 既存タスク（Todoistの状態を確認して同期）

具体的には、以下のようなイメージです。

```
タスク名: MTGの議事録をまとめる
---
notesフィールド:
詳細メモ...
[todoist:8765432109]  ← GASが自動で追記
```

notesに乗っかってしまう点は若干気持ち悪いですが、Google Tasks APIにはカスタムメタデータ用のフィールドがないため、現実的な妥協点として採用しています。

#### Google Tasksで手動削除されたタスクをどう検出するか

`PropertiesService`（GASのスクリプトプロパティ）に、前回同期時点でのtodoist_idの一覧を保存しておきます。
次回の同期時に「前回あったIDが今回のGoogle Tasksに存在しない」場合を手動削除と判定して、Todoist側も削除します。

```javascript
const prevIdsJson = props.getProperty(PREV_IDS_KEY);
const prevTodoistIds = prevIdsJson ? JSON.parse(prevIdsJson) : {};

// 同期処理後...
for (const [prevTodoistId, info] of Object.entries(prevTodoistIds)) {
  if (!currentTodoistIds[prevTodoistId]) {
    // Google Tasksから消えた → Todoistからも削除
    deleteTodoistTask(todoistToken, prevTodoistId);
  }
}
```

#### プロジェクト/リストの対応

Google Tasksの「リスト名」とTodoistの「プロジェクト名」が完全一致する場合にペアとして同期します。
一致しないものは、Google Tasksはデフォルトリスト、TodoistはInboxへ振り分けます。
また、Todoistに存在してGoogle Tasksにないプロジェクトがあれば、自動でGoogle Tasksにリストを作成するようにしています。

### 同期フローの詳細

実際の同期は3つのステップで動きます。

#### Todoist → Google Tasks（新規タスクの反映）

Todoistの全タスクを取得し、前回同期時のIDセットと比較します。
前回なかったIDが新規タスクとして検出され、対応するGoogle Tasksリストに作成されます。

この際、最初から`[todoist:xxx]`タグをnotesに付与することで、次のステップで二重登録されることを防いでいます。

また、TodoistのタスクはGoogle Tasks側では期日なしで作成しています。
Google TasksのAPIは期日超過のタスクを拒否する場合があり、Google Tasks側に期日管理を求めていないためです。

```javascript
function createGoogleTask(taskListId, todoistTask) {
  const idTag = `[todoist:${todoistTask.id}]`;
  const notes = todoistTask.description
    ? `${todoistTask.description}\n${idTag}`
    : idTag;
  const taskBody = {
    title: todoistTask.content,
    notes: notes,
    // 期日はGoogle Tasks側では管理しない
  };
  return Tasks.Tasks.insert(taskBody, taskListId).id;
}
```

#### Google Tasks → Todoist

Google Tasksの全リストを横断し、各タスクを処理します。

- `[todoist:xxx]`なし → Todoistに新規作成 → IDをGoogle Tasksのnotesに書き戻す
- `[todoist:xxx]`あり → Todoistのタスク状態を確認
  - 完了済み or 存在しない → Google Tasksから削除
  - 未完了 → 何もしない

#### 手動削除の検出

前回IDマップと今回の差分から、Google Tasksで手動削除されたタスクを検出してTodoistからも削除します。

なお、Google Tasksで手動完了されたタスクはTodoistに反映しません。
Todoistをマスターとしているため、完了はTodoist側で行う運用としています。

### APIトークンの管理について

Todoistのトークンはコードに直書きせず、GASの`PropertiesService`に格納しています。

```javascript
const props = PropertiesService.getScriptProperties();
const todoistToken = props.getProperty('TODOIST_API_TOKEN');
```

スクリプトプロパティはGASエディタの「プロジェクトの設定」から設定できます。
コードをGitHubなどで管理する場合もトークンが漏れません。

また、タスク作成時には`X-Request-Id`ヘッダーにUUIDを付与しています。
GASのトリガーが重複実行された場合の二重作成を防ぐ冪等性対策です。

```javascript
headers: {
  'Authorization': `Bearer ${token}`,
  'Content-Type': 'application/json',
  'X-Request-Id': Utilities.getUuid()
}
```

### Gemini連携の設定手順

セットアップの手順は以下のとおりです。

まず、GASエディタの左メニュー「サービス」から`Tasks API`を追加します。
![](/images/ai-task-consolidation-todoist/select-tasks-api.png)
追加後、以下のような設定になっていることを確認してください。
![](/images/ai-task-consolidation-todoist/setup-tasks-api.png)
これでGoogle Tasks APIがGASから使えるようになります。

次に、「プロジェクトの設定」→「スクリプトプロパティ」に以下を追加します。

| キー | 値 |
|------|----|
| `TODOIST_API_TOKEN` | Todoist設定 > 連携 > APIトークン |

設定完了後は以下のような状態になります。
![](/images/ai-task-consolidation-todoist/set-todoist-api-token-property.png)

設定が終わったら、`checkSetup()`関数を手動で一度実行してください。
Google Tasksのリスト一覧、Todoistのプロジェクト一覧、名前一致マッピング結果がログに出力されます。
期待通りのペアになっているか確認します。

最後に、`syncTasks()`関数に時間ベーストリガーを設定します。
5分ごとの実行を推奨しています。

### 同期スクリプトの全体コード

以下が実際に使っているGoogle Apps Scriptの全体コードです。

:::details Google Apps Script 全体コード

```javascript
// ============================================================
// 【セットアップ】
// 1. GASエディタ「サービス」→ Tasks API を追加
// 2. 「プロジェクトの設定」→「スクリプトプロパティ」に以下を追加
//    - TODOIST_API_TOKEN : TodoistのAPIトークン（Todoist設定 > 連携 > APIトークン）
// 3. syncTasks関数に時間ベースのトリガーを設定（例：5分ごと）
//
// 【リスト/プロジェクト対応ルール】
// - Google Tasksのリスト名 と Todoistのプロジェクト名 が完全一致する場合、そのペアで同期
// - 一致しない場合：Google Tasksはデフォルトリスト、TodoistはInbox
//
// 【双方向同期の仕様】
// Google Tasks → Todoist:
//   - notesに[todoist:xxx]がないタスクを新規としてTodoistに作成
//   - Google Tasksで手動削除されたタスクはTodoistからも削除
//
// Todoist → Google Tasks:
//   - PREV_TODOIST_TASK_IDS に存在しない新規タスクをGoogle Tasksに作成
//   - Todoistで完了/削除されたタスクはGoogle Tasksからも削除
//   - Todoistでプロジェクトが新規作成された場合、Google Tasksにも同名リストを作成
//
// 【二重登録防止】
// Todoistから同期したタスクはGoogle Tasks側のnotesに[todoist:xxx]タグを付与するため、
// 次回のGASサイクルで再びTodoistに登録されることはない
// ============================================================

const TODOIST_ID_PREFIX = '[todoist:';
const TODOIST_ID_SUFFIX = ']';
const PREV_IDS_KEY = 'PREV_TODOIST_IDS';               // { todoistId: { googleTaskId, googleListId } }
const PREV_TODOIST_TASK_IDS_KEY = 'PREV_TODOIST_TASK_IDS'; // Todoist側の全タスクID Set（JSON配列）

// ============================================================
// メイン関数（トリガーにこれを設定）
// ============================================================
function syncTasks() {
  const props = PropertiesService.getScriptProperties();
  const todoistToken = props.getProperty('TODOIST_API_TOKEN');
  if (!todoistToken) {
    Logger.log('ERROR: TODOIST_API_TOKEN が設定されていません');
    return;
  }

  // --- プロジェクト/リストのマッピングを構築 ---
  let googleTaskLists = getGoogleTaskLists();
  const todoistProjects = getTodoistProjects(todoistToken); // { name: projectId }

  // Todoistに存在してGoogle Tasksにないプロジェクトがあれば、Google Tasksにリストを作成
  googleTaskLists = syncProjectsToGoogleTaskLists(googleTaskLists, todoistProjects);

  // 名前が完全一致するペアを作成
  // { googleTaskListId: { todoistProjectId, listName } }
  const listMapping = buildListMapping(googleTaskLists, todoistProjects);

  Logger.log('リスト対応マップ:');
  for (const [gtListId, info] of Object.entries(listMapping)) {
    Logger.log(`  "${info.listName}" → Todoist Project: ${info.todoistProjectId || 'Inbox'}`);
  }

  // --- 前回同期時のIDマップを取得 ---
  const prevIdsJson = props.getProperty(PREV_IDS_KEY);
  const prevTodoistIds = prevIdsJson ? JSON.parse(prevIdsJson) : {};

  const prevTodoistTaskIdsJson = props.getProperty(PREV_TODOIST_TASK_IDS_KEY);
  const prevTodoistTaskIds = new Set(prevTodoistTaskIdsJson ? JSON.parse(prevTodoistTaskIdsJson) : []);

  // --- 今回のIDマップ ---
  const currentTodoistIds = {};         // Google Tasks起点で管理: { todoistId: { googleTaskId, googleListId } }
  const currentTodoistTaskIds = new Set(); // Todoist全タスクID

  // ============================================================
  // STEP 1: Todoist → Google Tasks 同期
  //   Todoistの全タスクを取得し、新規タスクをGoogle Tasksに反映
  // ============================================================
  const allTodoistTasks = getAllTodoistTasks(todoistToken);

  // プロジェクトIDからGoogleリストIDへの逆引きマップ
  // { todoistProjectId: googleTaskListId }
  const projectToListMap = buildProjectToListMap(listMapping);

  for (const todoistTask of allTodoistTasks) {
    currentTodoistTaskIds.add(todoistTask.id);

    if (!prevTodoistTaskIds.has(todoistTask.id)) {
      // 前回存在しなかった = Todoistで新規作成されたタスク
      // Google Tasksの対応リストに作成
      const targetGtListId = projectToListMap[todoistTask.project_id] || getDefaultGoogleTaskListId(googleTaskLists);
      const newGtTaskId = createGoogleTask(targetGtListId, todoistTask);
      if (newGtTaskId) {
        currentTodoistIds[todoistTask.id] = { googleTaskId: newGtTaskId, googleListId: targetGtListId };
        Logger.log(`Google Tasks新規作成（Todoistから）: "${todoistTask.content}" → GT ID: ${newGtTaskId}`);
      }
    }
  }

  // ============================================================
  // STEP 2: Google Tasks → Todoist 同期
  //   Google Tasksの全タスクを取得し、新規/既存を処理
  // ============================================================
  for (const [gtListId, info] of Object.entries(listMapping)) {
    const googleTasks = getGoogleTasks(gtListId);

    for (const task of googleTasks) {
      const existingTodoistId = extractTodoistId(task.notes);

      if (!existingTodoistId) {
        // 新規タスク（todoist_idなし）→ Todoistに作成
        const newTodoistId = createTodoistTask(todoistToken, task, info.todoistProjectId);
        if (newTodoistId) {
          appendTodoistIdToTask(gtListId, task.id, task.notes, newTodoistId);
          currentTodoistIds[newTodoistId] = { googleTaskId: task.id, googleListId: gtListId };
          currentTodoistTaskIds.add(newTodoistId);
          Logger.log(`Todoist新規作成（Google Tasksから）: "${task.title}" → Todoist ID: ${newTodoistId}`);
        }
      } else {
        // 既存タスク（todoist_idあり）→ Todoist側の状態を確認
        const todoistTask = getTodoistTask(todoistToken, existingTodoistId);
        currentTodoistIds[existingTodoistId] = { googleTaskId: task.id, googleListId: gtListId };

        if (!todoistTask || todoistTask.is_deleted || !!todoistTask.completed_at) {
          // Todoist側で完了 or 削除 → Google Tasksから削除
          deleteGoogleTask(gtListId, task.id);
          delete currentTodoistIds[existingTodoistId];
          currentTodoistTaskIds.delete(existingTodoistId);
          Logger.log(`Google Tasks削除（Todoist完了済み）: "${task.title}"`);
        }
        // Todoist未完了の場合は何もしない
      }
    }
  }

  // ============================================================
  // STEP 3: Google Tasksで手動削除されたタスクをTodoistからも削除
  // ============================================================
  for (const [prevTodoistId, info] of Object.entries(prevTodoistIds)) {
    if (!currentTodoistIds[prevTodoistId]) {
      deleteTodoistTask(todoistToken, prevTodoistId);
      currentTodoistTaskIds.delete(prevTodoistId);
      Logger.log(`Todoist削除（Google Tasks手動削除）: Todoist ID: ${prevTodoistId}`);
    }
  }

  // --- 今回のIDマップを保存 ---
  props.setProperty(PREV_IDS_KEY, JSON.stringify(currentTodoistIds));
  props.setProperty(PREV_TODOIST_TASK_IDS_KEY, JSON.stringify([...currentTodoistTaskIds]));
}

// ============================================================
// プロジェクト/リスト同期
// ============================================================

// Todoistプロジェクトに存在してGoogle Tasksにないリストを作成
function syncProjectsToGoogleTaskLists(googleTaskLists, todoistProjects) {
  const existingListNames = new Set(googleTaskLists.map(l => l.title));
  for (const projectName of Object.keys(todoistProjects)) {
    if (!existingListNames.has(projectName)) {
      const newList = createGoogleTaskList(projectName);
      if (newList) {
        googleTaskLists.push(newList);
        Logger.log(`Google Tasksリスト作成（Todoistプロジェクトから）: "${projectName}"`);
      }
    }
  }
  return googleTaskLists;
}

function buildListMapping(googleTaskLists, todoistProjects) {
  const mapping = {};
  for (const gtList of googleTaskLists) {
    const matchedProjectId = todoistProjects[gtList.title] || null;
    mapping[gtList.id] = {
      listName: gtList.title,
      todoistProjectId: matchedProjectId
    };
  }
  return mapping;
}

// { todoistProjectId: googleTaskListId } の逆引きマップ
function buildProjectToListMap(listMapping) {
  const map = {};
  for (const [gtListId, info] of Object.entries(listMapping)) {
    if (info.todoistProjectId) {
      map[info.todoistProjectId] = gtListId;
    }
  }
  return map;
}

function getDefaultGoogleTaskListId(googleTaskLists) {
  // 先頭リストをデフォルトとして使用
  return googleTaskLists.length > 0 ? googleTaskLists[0].id : '@default';
}

// ============================================================
// Google Tasks API
// ============================================================

function getGoogleTaskLists() {
  try {
    const response = Tasks.Tasklists.list({ maxResults: 100 });
    return response.items || [];
  } catch (e) {
    Logger.log(`Google Tasksリスト取得エラー: ${e}`);
    return [];
  }
}

function createGoogleTaskList(name) {
  try {
    return Tasks.Tasklists.insert({ title: name });
  } catch (e) {
    Logger.log(`Google Tasksリスト作成エラー: ${e}`);
    return null;
  }
}

function getGoogleTasks(taskListId) {
  try {
    const response = Tasks.Tasks.list(taskListId, {
      showCompleted: false,
      showHidden: false,
      maxResults: 100
    });
    return response.items || [];
  } catch (e) {
    Logger.log(`Google Tasks取得エラー: ${e}`);
    return [];
  }
}

// TodoistタスクをGoogle Tasksに作成（[todoist:xxx]タグをnotesに付与して二重登録防止）
function createGoogleTask(taskListId, todoistTask) {
  const idTag = `${TODOIST_ID_PREFIX}${todoistTask.id}${TODOIST_ID_SUFFIX}`;
  const notes = todoistTask.description
    ? `${todoistTask.description}\n${idTag}`
    : idTag;

  const taskBody = {
    title: todoistTask.content,
    notes: notes,
  };

  try {
    const result = Tasks.Tasks.insert(taskBody, taskListId);
    return result.id;
  } catch (e) {
    Logger.log(`Google Tasksタスク作成エラー: ${e}`);
    return null;
  }
}

function deleteGoogleTask(taskListId, taskId) {
  try {
    Tasks.Tasks.remove(taskListId, taskId);
  } catch (e) {
    Logger.log(`Google Tasks削除エラー: ${e}`);
  }
}

function appendTodoistIdToTask(taskListId, taskId, currentNotes, todoistId) {
  const idTag = `${TODOIST_ID_PREFIX}${todoistId}${TODOIST_ID_SUFFIX}`;
  const newNotes = currentNotes ? `${currentNotes}\n${idTag}` : idTag;
  try {
    Tasks.Tasks.patch({ notes: newNotes }, taskListId, taskId);
  } catch (e) {
    Logger.log(`Google Tasks更新エラー: ${e}`);
  }
}

// ============================================================
// Todoist API
// ============================================================

function getTodoistProjects(token) {
  try {
    const response = UrlFetchApp.fetch('https://api.todoist.com/api/v1/projects', {
      method: 'GET',
      headers: { 'Authorization': `Bearer ${token}` },
      muteHttpExceptions: true
    });
    if (response.getResponseCode() === 200) {
      const result = JSON.parse(response.getContentText());
      const projects = result.results;
      const map = {};
      for (const project of projects) {
        map[project.name] = project.id;
      }
      return map;
    } else {
      Logger.log(`Todoistプロジェクト取得エラー: ${response.getResponseCode()}`);
      Logger.log(`Todoistプロジェクト取得エラー: ${response.getContentText()}`);
      return {};
    }
  } catch (e) {
    Logger.log(`Todoistプロジェクト取得例外: ${e}`);
    return {};
  }
}

// Todoistの全タスクを取得
function getAllTodoistTasks(token) {
  try {
    const response = UrlFetchApp.fetch('https://api.todoist.com/api/v1/tasks', {
      method: 'GET',
      headers: { 'Authorization': `Bearer ${token}` },
      muteHttpExceptions: true
    });
    if (response.getResponseCode() === 200) {
      const result = JSON.parse(response.getContentText());
      return result.results || [];
    } else {
      Logger.log(`Todoist全タスク取得エラー: ${response.getResponseCode()} ${response.getContentText()}`);
      return [];
    }
  } catch (e) {
    Logger.log(`Todoist全タスク取得例外: ${e}`);
    return [];
  }
}

function createTodoistTask(token, googleTask, projectId) {
  const payload = {
    content: googleTask.title,
  };
  if (googleTask.notes) {
    const cleanNotes = removeTodoistIdTag(googleTask.notes);
    if (cleanNotes.trim()) {
      payload.description = cleanNotes.trim();
    }
  }
  if (googleTask.due) {
    payload.due_date = googleTask.due.date;
  }
  if (projectId) {
    payload.project_id = projectId;
  }

  try {
    const response = UrlFetchApp.fetch('https://api.todoist.com/api/v1/tasks', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
        'X-Request-Id': Utilities.getUuid()
      },
      payload: JSON.stringify(payload),
      muteHttpExceptions: true
    });
    if (response.getResponseCode() === 200) {
      const task = JSON.parse(response.getContentText());
      return task.id;
    } else {
      Logger.log(`Todoist作成エラー: ${response.getResponseCode()} ${response.getContentText()}`);
      return null;
    }
  } catch (e) {
    Logger.log(`Todoist作成例外: ${e}`);
    return null;
  }
}

function getTodoistTask(token, todoistId) {
  try {
    const response = UrlFetchApp.fetch(`https://api.todoist.com/api/v1/tasks/${todoistId}`, {
      method: 'GET',
      headers: { 'Authorization': `Bearer ${token}` },
      muteHttpExceptions: true
    });
    if (response.getResponseCode() === 200) {
      return JSON.parse(response.getContentText());
    } else if (response.getResponseCode() === 404) {
      return null;
    } else {
      Logger.log(`Todoist取得エラー: ${response.getResponseCode()}`);
      return null;
    }
  } catch (e) {
    Logger.log(`Todoist取得例外: ${e}`);
    return null;
  }
}

function deleteTodoistTask(token, todoistId) {
  try {
    const response = UrlFetchApp.fetch(`https://api.todoist.com/api/v1/tasks/${todoistId}`, {
      method: 'DELETE',
      headers: { 'Authorization': `Bearer ${token}` },
      muteHttpExceptions: true
    });
    if (response.getResponseCode() !== 204) {
      Logger.log(`Todoist削除エラー: ${response.getResponseCode()}`);
    }
  } catch (e) {
    Logger.log(`Todoist削除例外: ${e}`);
  }
}

// ============================================================
// ユーティリティ
// ============================================================

function extractTodoistId(notes) {
  if (!notes) return null;
  const regex = /\[todoist:([^\]]+)\]/;
  const match = notes.match(regex);
  return match ? match[1] : null;
}

function removeTodoistIdTag(notes) {
  if (!notes) return '';
  return notes.replace(/\n?\[todoist:[^\]]+\]/g, '');
}

// ============================================================
// 初回セットアップ確認用（手動で一度実行して確認）
// ============================================================
function checkSetup() {
  const props = PropertiesService.getScriptProperties();
  const token = props.getProperty('TODOIST_API_TOKEN');
  Logger.log('TODOIST_API_TOKEN: ' + (token ? '設定済み' : '未設定'));
  if (!token) return;

  // Google Tasksリスト一覧
  try {
    const lists = Tasks.Tasklists.list();
    Logger.log('\n--- Google Tasks リスト ---');
    (lists.items || []).forEach(l => Logger.log(`  "${l.title}" (ID: ${l.id})`));
  } catch (e) {
    Logger.log('Google Tasks API エラー: サービスが有効化されていない可能性があります');
  }

  // Todoistプロジェクト一覧
  const projects = getTodoistProjects(token);
  Logger.log('\n--- Todoist プロジェクト ---');
  for (const [name, id] of Object.entries(projects)) {
    Logger.log(`  "${name}" (ID: ${id})`);
  }

  // マッチング結果
  const googleTaskLists = getGoogleTaskLists();
  const mapping = buildListMapping(googleTaskLists, projects);
  Logger.log('\n--- 名前一致マッピング結果 ---');
  for (const [gtListId, info] of Object.entries(mapping)) {
    const dest = info.todoistProjectId ? `Todoist: "${info.listName}"` : 'Todoist: Inbox（一致なし）';
    Logger.log(`  Google Tasks: "${info.listName}" → ${dest}`);
  }
}
```

:::

### 実装のポイント

コードの中で特に意識した点をいくつか補足します。

#### 二重登録の防止

Todoistから同期したタスクには、Google Tasks側のnotesに`[todoist:xxx]`タグが付与されます。
このタグがあるタスクは、次回のGASサイクルで「既存タスク」として扱われるため、再びTodoistに登録されることはありません。
逆方向も同様で、Google TasksからTodoistに作成した直後にIDを書き戻すことで、次のサイクルでの重複を防いでいます。

#### 初回実行時の挙動

初回実行時は`PREV_TODOIST_TASK_IDS`が空のため、Todoistの既存タスクが全件Google Tasksに作成されます。
許容できない場合は、事前にTodoistの既存タスクIDをプロパティに書き込む初期化処理が別途必要になります。

#### ラベル・優先度の同期について

Google Tasksにはラベルや優先度に相当するフィールドがAPIで扱えないため、これらはTodoist側で手動設定する運用としています。
notesにフォーマットを設けてパースする案も検討しましたが、Geminiが毎回そのフォーマットで書いてくれる再現性が期待できないため断念しました。

### Gemini連携の制約

#### 直接連携ではない

中継が必要なため、構成が一段複雑になります。

#### 厳密な同期は難しい

リアルタイム連携ではないため、即時反映は期待しにくいです。

#### Google Tasksのイベントをそのままトリガーにしづらい

タスクの作成・編集・削除、これらを即時トリガーとして扱うことが難しいです。
Google Tasks APIにはWebhookのような仕組みがないため、イベント駆動での処理が組みにくいのです。

#### そのため、定期実行ベースの同期になる

5分間隔などでチェックし、定期的に整合性を見て、差分を反映する形になります。

#### Google Tasks経由では、Todoistの情報を完全には再現しづらい

少なくとも自分の運用では、ラベルや優先度をうまく再現性ある形で扱えませんでした。
タスクタイトルや説明、期限などは扱えても、Todoist固有の整理情報は落ちることがあります。
そのため、Gemini連携ではラベルや優先度の再現はある程度諦めています。

#### 結果として、ClaudeやChatGPTより運用上のクセがある

同じ感覚では使えません。タイムラグを前提にする必要がありますし、情報の欠落を許容した上で使う必要があります。

### それでもGeminiを組み込みたい理由

普段のワークフロー上、Geminiを無視できませんし、資料作成系のタスクを取りこぼしたくないのです。
完璧ではなくても、集約先に寄せる価値はあると考えています。

## 実際に運用してみて感じていること

### 一番大きいのは、タスク入力の負担が減ったこと

自分で整えて入力する回数が減りました。
とりあえずAIに投げればよくなったのは大きいです。
「あとで登録しよう」と思って結局やらなかった、ということが減りました。

### タスク整理の心理的ハードルが下がった

「ちゃんと書かなきゃ」という気持ちが減りました。
雑でも残せるし、会話から自然に残せます。
この心理的ハードルの低下は、想像以上に効果がありました。

### 完全に自動化されているわけではない

微調整は必要ですし、連携ごとにクセもあります。
とくにGeminiは制約が大きいので、すべてが勝手にうまくいくわけではありません。

### それでも、自分で全部管理するよりはかなり楽

100点ではありませんが、続けやすいです。
現実的な改善としてはかなり満足しています。

## この記事で伝えたい注意点

### これは"唯一の正解"ではない

この記事は、生成AI連携に最適なタスクツールの決定版ではなく、あくまで自分の運用例です。

### Todoistを選んだのは、主に使い慣れていたから

ベストツール選定の結果ではなく、自分が日常で使えることを優先しました。

### 生成AIごとに連携の質は揃わない

Claude、ChatGPT、Gemini、それぞれ連携の深さや精度が違うので、同じレベルの連携は難しいです。

### とくにGeminiは制約を理解した上で使う必要がある

直接連携ではないこと、タイムラグがあること、ラベルや優先度の再現が難しいこと、仕組みがやや重いこと、これらを理解した上で使う必要があります。

## それでも、同じ悩みを持つ人には参考になると思う

### タスク管理を丁寧に回すのが苦手な人へ

自分で全部背負わなくてもいいと思います。
生成AIに寄せるという選択肢があることを知ってもらえたら嬉しいです。
自分はそれでかなり楽になりました。

### 複数の生成AIを使っている人へ

AIごとにタスクが散る問題は起こりやすいです。
だからこそ、受け皿を決める意味があります。
どこに入れたか分からなくなる前に、集約先を決めておくと安心感があります。

### 完全一元化でなくても、十分価値はある

できる範囲で寄せるだけでも違いますし、それだけでも現実的な運用改善になります。
完璧を目指す必要はないと思っています。

## おわりに

自分は、タスク管理を頑張るより、生成AIに寄せるほうが合っていました。
Claudeから始まり、ChatGPTやGeminiにも広げてきました。
Todoistをマスターにすることで、かなり見通しはよくなりました。

まだ制約はあります。
とくにGemini連携は仕組みとしてまだ改善の余地があります。
ただ、生成AIの進化は早いので、今後の連携のしやすさにも期待しています。

同じように「可能な限り一元管理したい」と思っている人の参考になればうれしいです。
ここまで読んでいただきありがとうございました。
