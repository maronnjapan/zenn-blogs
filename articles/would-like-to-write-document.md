---
title: "「そこに全てが書いてある」を目指して"
emoji: "✨"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["document"]
published: true
---

## はじめに

皆さんはドキュメントなど書いていますか？
私は書いたり、書かなかったりです。
ただ、最近ドキュメントを残すことって大事だなと感じたので、今回はその想いを綴ったものとなります。
なお、ここでいうドキュメントはプロジェクトの概要やその価値などユーザー利益に関わるものではなく、内部の設計であったり、実装手順など開発者のために書かれたものを想定しております。

## この記事のここだけ!!!

ドキュメント残すの頑張ります。

## 「なんでも聞いて」は罠

ドキュメントなどを残しておきたいと思った最大の理由はここにあります。
「なんでも聞いて」はそのままの意味で使用されることはないと思います。
具体的にいうと、「なんでも聞いて」は「**調べてわからなかったことは**なんでも聞いて」という意味を暗に含んでいる認識です。
そのため、実際質問するのは以下の流れになります。
① 不明点に出会う
② ネットや書籍で調べる
③ 分からない
④ 調べた内容を記載しつつ、質問する
質問する流れとしては自然な気がします。
しかし、① から ③ に到達する流れは本来不要である可能性があります。
すでに解消方法が分かっていることや、プロジェクト固有の話であれば調べるより聞いた方が早いからです。
質問すれば無駄に時間を使う必要がなく、効率的に業務をこなせます。
一方、質問をせずに一人で悩んでしまい、すぐ解決できるものに時間をかけてしまうと本来見込めた進捗を遅らせてしまいます。
ということがあるので、30 分悩んで分からなかったら質問するといったことができたり、できる人というのは質問が上手いと言われたりします。
このように質問をきちんとするというのは大切です。
なので、適切な質問のタイミングを学び、業務の効率化を目指すべきです。
ですが、この質問のタイミングを個人の努力だけで片付けてしまうのも良くないと考えます。
質問を受ける側として、悩むべき話なのか判断基準を設けておくことも同様に大切だと思っています。
判断基準がない場合、聞くべき質問なのか自分で調べるべきことなのかを個人が選択できてしまいます。
一方で、以下のような決まりごとが浸透していた場合について見ていきます。
① 疑問が発生する。
② プロジェクトのドキュメントを検索する。
③ 欲しい情報にたどり着けない場合、質問する or ドキュメントに存在しないことを報告し調査することを伝える。
④ 解決策が見つかる
⑤ 記録へ残す
このように行う文化がチームの共通認識になっていれば、既知の問題について悩めてしまうことは少なくなります。
例に挙げたルールが適切か、ドキュメントも検索しやすい状態であるかなど詳細については議論すべきかと思いますが、少なくとも既知の問題を悩めてしまうという仕組み上の問題は少なくできると思っています。
私はこのような状況にしたく、その土台としてドキュメントの充実があると考えます。

## どういった内容で記録に残すのか？

書く内容としては以下のものでも良いと考えます。

```
○○を××したい場合は、～さんに質問すること。
```

人に聞かないと分からないということしか分からないですが、そのプロジェクトで話題には挙がっており、解決方法への道筋が提示されています。
この道筋が見えていることが、文字に残しておく価値だと思います。
このケースの価値は、質問するための負荷を下げることにあります。
まず、質問する人が明確なので分かる人を探すための時間が少なくなります。
そして、質問するときも 1~10 まで書く必要がなくなり、記載内容のリンクと共に「ここに書いてあることがしたいです」と言うだけでお互いに何がしたいかの合意を取ることができます。
質問者が自分で考えるか、人に聞くかを悩む時間も減らせます。
これだけでも、ドキュメントを残しておく価値があります。
ただ、流石にこのままでは連絡網の役割しかないので、質問者もしくは回答者が都度追記していくことが望ましいですが、残す内容は何でも良いと考えます。
大事なのは、そこに情報があるはずだという共通認識が生まれることなので、内容をしっかり書かないといけないと思い、書くことそのものをやめてしまうよりは、徐々に内容を完成させていくつもりでとりあえず記録を残すことを望みます。

## 何を記録に残すのか？

内容は何でもいい、とにかく記録することが大切だと書きましたが、そもそもどのような事象をドキュメントに残すべきかは考える必要があります。
私の考えは以下の二つを記載すべきだと思っています。

- プロジェクト固有の実装手順
- Qiita や Zenn、個人ブログで欲しい情報が見つからない。
  それぞれ見ていきます。

### 残すべき事象 ①: プロジェクト固有の実装手順

プロジェクト固有のものは全て残すべきだと思います。
ここでいう固有は、コーディング規約だけではなく、統一させたい実装手順を含みます。
統一させたい実装手順についての例を上げます。
BFF とバックエンドを NestJS で構築し、両者を通信させる実装を行っているとします。
この時は、使用している技術と実装手順を記載すべきです。
仮に gRPC で通信をしているのに、そのことを記載していないと新しく入ってきた人は REST で通信しようとする可能性があります。
これは大きな手戻りを発生させます。
流石に、上記のようなことは発生しにくいですが、gRPC で通信するという事実を残しておくだけでは不十分です。
NestJS での gRPC 通信のクライアント側は、[ドキュメント](https://docs.nestjs.com/microservices/grpc#client)を確認すると`ClientsModule` を使用しています。
しかし、`ClientsModule`を使わずとも通信することは可能です。
そのため、何かしらの手順を確立しておかないと様々な手段で通信が行われてしまい、よく分からないけど特定の機能だけは値が取得できるという状態が生まれます。
上記の場合を避けるため、プロジェクトでは`ClientsModule`を用いて通信しますと決めておくことは自然な流れです。
この取り決めたことこそ、統一させたい実装手順となります。
まとめると、複数の手段があるがその中で特定の手段にするとプロジェクトで定めたものもドキュメントなどに残しておくべきだと考えます。

### 残すべき事象 ②: Qiita や Zenn、個人ブログで欲しい情報が見つからない。

この場合は、当該事象について広く扱われている内容ではなく、情報へのアクセスがしにくい状況を指しています。
そのため、他の人が再度同じ問題に遭遇した時に解消時間が長くなります。
ただ、該当の問題は以前解決されているので、もう一度探すことは不必要な時間となります。
参考となる情報を再度自分の中で嚙み砕くことは理解へとつながりますが、解決策があるのに探すための時間を設けることは大きな価値をもたらすとは考えにくいです。
以上のような、技術的な話でプロジェクト固有というほどではないが、情報量が少なく必要な情報にアクセスしにくいことも文章として残しておくべきだと思います。
一方で、Javascript の map 関数の使い方といった、調べればいくらでも出てくる内容を記録することは正直無駄だと思っています。
その線引きとして、日本語での情報量を挙げています。

## ドキュメントを残さなくても良い場合

全てのプロジェクトにおいて、ドキュメントをきっちり残すことは正しいと言えません。
まず一人で実装などを行っており、今後人が追加されることがほぼない場合です。
この場合は、本人が把握さえできれば良いので、ドキュメントをきっちり書く労力の割に読む人はいません。
基本的にプロジェクト内のドキュメントは特定の情報を知りたい人しか読みません。
そのため、読む人=書く人の時にはドキュメントの存在意義はそこまでありません。
そして、追加で開発者が増える機会もそこまでないなら、入ってきたタイミングで都度伝える形でもそれほど時間はとられないでしょう。
ドキュメントを残す意義は、価値を届けための時間の増大を生み出すことです。
すなわち、ドキュメントを残すことが価値を届けるための時間の増大につながらい場合はドキュメントを残すのはむしろ悪影響を及ぼします。

## 価値を届けるための時間をお互い増やすために

ここまで記録を残すことを考えてきました。
そのため、中には同期コミュニケーションを排除して、非同期コミュニケーションだけでも業務が回るようにしていきたいと感じる方がいらっしゃるかもしれません。
というより、むしろそういう風に思ってもらえるようにここまで書いてきたつもりです。
ただ、私は同期コミュニケーションを排除したいわけではありません。
むしろ、同期コミュニケーションの価値を高めるために非同期コミュニケーションの充実を目指しています。
私としては、非同期でも解決できる問題は非同期で解決して、同期コミュニケーションのための時間を空けておくことが大切だと思っています。
チームメンバーの理解であったり、価値を届けるための検討など同期コミュニケーションでしかなしえないことは多くあります。
一方で、非同期でしか解決できない問題というのはそれ程多くありません。
証拠を残す意味合いで、文章を使ったやり取りを行うことはありますが、価値の創出という観点では非同期でしか解決できない事象は思い浮かびません。
このように同期コミュニケーションは可能性の塊で、価値がとっても高いです。
その同期コミュニケーションを非同期コミュニケーションでも解決できることで、時間をとってしまうのは非常にもったいないです。
価値を生み出す同期コミュニケーションの時間を多く確保するためにも、非同期コミュニケーションの下地を充実させたいと思い、決意表明のような形でこの記事を書きました。
（追記: この記事を草案を書ききった後に、桜井さんの[内圧をカンカンに高める 【仕事の姿勢】](https://www.youtube.com/watch?v=X6FcraeKrSM)という動画を目にしてしまいました。とても勉強になるなと思ったのと同時に、この記事は圧を抜いてしまうのでは？とも感じてしまいました。ただ、せっかく書いたので今回は投稿しちゃいました）

## おわりに

今回はドキュメントについて、自身が思ったことや今後頑張っていくぞということを書きました。
「[GitLab に学ぶ 世界最先端のリモート組織のつくりかた ドキュメントの活用でオフィスなしでも最大の成果を出すグローバル企業のしくみ](https://www.seshop.com/product/detail/25794?gad_source=1&gclid=CjwKCAiA_aGuBhACEiwAly57MfJK7e_4WFtTdEIvrk0bQ7zqaD9b5C524LjLnI1qFn1LNbauSTy7MRoC1q8QAvD_BwE)」や「[人が増えても速くならない ～変化を抱擁せよ～](https://gihyo.jp/book/2023/978-4-297-13565-2)」とかを読んで、いいこと書いてあるなと思い、自身の関心毎の一つであるドキュメントと絡めて書きました。
ただ、この記事は未完成です。
一つは、上記二つの書籍の内容であったり、書きたいなと思っていることの多くが表現できていないからです。
特に「[人が増えても速くならない ～変化を抱擁せよ～](https://gihyo.jp/book/2023/978-4-297-13565-2)」の 2 章 1 節「•2 倍の予算があっても 2 倍の生産性にはならない」部分については、とても腑に落ちて感心したので、何とかここの内容を盛り込みたいとおもったのですが、上手くできませんでした。
私の文章力のなさで、上記の魅力を伝えられないのが歯がゆいので、是非「[人が増えても速くならない ～変化を抱擁せよ～](https://gihyo.jp/book/2023/978-4-297-13565-2)」を読んでください。
未完成である理由の二つ目は、まだ決意表明の段階ということです。
ちょこちょこ文章には残しているつもりですが、それが実際にどう役に立ったかまでは実践できておりません。
そのため、やってみると修正を余儀なくされる可能性がかなりあります。
この記事が机上の空論となるかどうかは今後にかかっているため、まだ未完成だと思っています。
この先、文章力も上がりドキュメントの成熟を行っていくことで、自身の考えがどうであったかを見つめていきたいと思います。
ここまで読んでいただきありがとうございました。
