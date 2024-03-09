---
title: "Claudeが発表した新モデルについての記事の一部抜粋と感想"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AI", "claude"]
published: true
---

## はじめに

生成 AI で有名な Claude が先日新モデルを提供しました。
今回はその Claude が投稿した[新モデルについての記事](https://www.anthropic.com/news/claude-3-family)の一部を抜粋し、個人的な感想を少し差し込んでいます。
原文も短いので、この記事で内容が気になると思ったら是非以下のリンクから飛んでみてください。
[https://www.anthropic.com/news/claude-3-family](https://www.anthropic.com/news/claude-3-family)

## 賢さの向上

生成 AI の一つである、Claude が提供する 3 モデルが多くの項目で向上をしたという発表がありました。
具体的には以下のベンチマークとなっています。
![**[Introducing the next generation of Claude](https://www.anthropic.com/news/claude-3-family) より引用**](/images/new-release-article-in-claude/Untitled.png)
**[Introducing the next generation of Claude](https://www.anthropic.com/news/claude-3-family) より引用**
Claude が発表していることもあり、どの項目を Claude の最も賢いモデルがトップになっています。
特に推論・コードの評価・多言語対応について大きな強みを発揮していそうです。
ただ、他は GPT-4 や Gemin 1.0 Pro と比べて多少優れているぐらいです。

## 素早いレスポンス

[こちらの項目](https://www.anthropic.com/news/claude-3-family#:~:text=Near%2Dinstant%20results)では質問から回答までの速度が大きく向上したと言及しています。
特に Claude3 の Haiku モデルは最速かつ、コストも低く実行できるようです。
具体例として、以下のように言及しています。

> It can read an information and data dense research paper on arXiv (~10k tokens) with charts and graphs in less than three seconds.

[arXiv](https://arxiv.org/)という様々な論文が保存・公開されているウェブサイトの情報であれば、グラフやチャート含めても 3 秒より短い時間で読み込めるそうです。
すごいですね。
Sonnet モデルも Claude2 や Claude2.1 に比べて 2 倍ほと早くなったそうです。
Opus モデルは Claude2 や Claude2.1 と同程度の速度だそうです。
ただ、両者共により賢くなっており、Opus モデルは格段に賢くなったと言及しています。

## 視覚情報処理の向上

以下表のように、写真・グラフ・チャート・図での読み取り能力についても向上を確認できたそうです。
![**[Introducing the next generation of Claude](https://www.anthropic.com/news/claude-3-family) より引用**](/images/new-release-article-in-claude/2024-03-09_14h43_39.png)
**[Introducing the next generation of Claude](https://www.anthropic.com/news/claude-3-family) より引用**
個人的にはどれも横ばいかなという印象は受けます。
ただ、図の読み取りについては少し Claude が強いのかなと感じます。

## 回答拒否の減少

モデルの性能向上で、以前に比べてプロンプトのニュアンスや回答することが適切でないかの判断を正確に判断できるようになったとのことです。
これによって、以下グラフのように特に問題のない質問に対する回答を拒否する割引が減少されたとのことです。
![**[Introducing the next generation of Claude](https://www.anthropic.com/news/claude-3-family) より引用**](/images/new-release-article-in-claude/2024-03-09_14h51_49.png)
**[Introducing the next generation of Claude](https://www.anthropic.com/news/claude-3-family) より引用**

## Opus モデルの正確性向上

かつてのモデルが弱点としている複雑かつ事実に基づく質問を行った結果、以下のように 2 倍正しい回答を出力するようになっています。
![2024-03-09_14h58_55.png](/images/new-release-article-in-claude/2024-03-09_14h58_55.png)
一方で、正しくない答えや「分からない」と答える割合が減少し、Opus モデルの回答が以前より正確になっていることを示しています。
個人的には Sonnet モデルや Haiku モデルの結果もみたいですね。

## **Long context and near-perfect recall**

ここは私の知識不足でいまいち何が向上したのかよくわかりませんでした。すみません。
なので、気になる方は[原文](https://www.anthropic.com/news/claude-3-family#:~:text=Long%20context%20and%20near%2Dperfect%20recall)を確認お願いします。

## 安全に配慮した作り

性能の向上だけでなく、安全性やプライバシーなどに配慮して作っていることを強調していました。
この中で、[Constitutional AI](https://www.anthropic.com/news/constitutional-ai-harmlessness-from-ai-feedback)という単語があることをはじめて知りました。

## **Easier to use**

ここは省略します。

## **Model details**

ここもこれまで読んだ内容をまとめている部分がほとんどだったので、省略します。

## いつごろから使えるようになるか

この記事が投稿された 2024/03/04 時点で、Opus と Sonnet モデルは使用できるそうです。
ただ、Opus モデルは Pro プランを契約する必要があります。
また、Sonnet モデルは[Amazon Bedrock](https://aws.amazon.com/jp/bedrock/)や[Google Cloud’s Vertex AI Model Garden](https://cloud.google.com/model-garden?hl=ja)経由でも使用できるようです。

## おわりに

今回は Claude の新モデルのリリース記事をまとめたものでした。
GPT-4 がすごいと言っていたら、それを超えるものできたよというものがすぐに上がります。
熱い分野なんだなと改めて感じるのと、使いこなせていないことに不安を覚えるくらいのスピード感があります。
とはいえ、これだけ新しいことで盛り上がるのも楽しいですね。
ここまで読んでいただきありがとうございました。
