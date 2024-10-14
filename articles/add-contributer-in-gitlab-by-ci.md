---
title: "GitLabでCIを使って、READMEに疑似コントリビューター一覧項目を設ける機能を搭載してみた"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitLab","CI"]
published: true
---
## はじめに
業務の一環でGitLabを使用しており、そこでGit管理したライブラリを置いています。
このGitLabは、コントリビューターの設定はできるですが、それをプロジェクトのTOPページで表示させることはできません。(できたら申し訳ございません。あくまで、私が確認した範囲では表示する方法がわかりませんでした。)
一方、GitHubはプロジェクトのトップページに、以下のようなコントリビューターを表示させる項目があります。
![2024-10-14_11h53_25.png](/images/add-contributer-in-gitlab-by-ci/2024-10-14_11h53_25.png)
GitHubのように、コントリビューターをプロジェクトのトップで表示させたかったのが、この記事を書くきっかけです。
完全なコントリビューターの仕組みは構築できていませんが、それっぽいものはできたので、共有します。
## そもそもなぜこの機能を搭載しようと思ったのか
コードを展開する前に、なぜこの機能を搭載しようと思ったのかについても記載します。
具体的には以下の二つです。
- マージリクエストを出してくれる人に感謝を示したかった
- プロジェクト内のコードを触ったことがある人を可視化したかった
それぞれ簡単に説明します。
### 動機①：マージリクエストを出してくれる人に感謝を示したかった
これが、全体の7~8割ほど占めます。
作っている側としては、便利であり、要件を効率的に達成できるようにすることを目的として、ライブラリを提供しています。
しかし、使用する側は必ずしもそのライブラリでないと、ダメということはまずありません。
そのような状況で、自分たちが作ったライブラリを使ってもらえるのはありがたい限りです。
さらに、内部のコードを改修して、マージリクエストを出していただけるのは、相当嬉しいです。
これら感謝を示すために、プロジェクトへアクセスした時、目に付く形でプロジェクトの発展貢献いただいた方を、表示したいと思いました。
パッと目に付くのはどこかと考えた時に、READMEの先頭に表示されれば達成できると思い、調べるに至りました。
### 動機②：プロジェクト内のコードを触ったことがある人を可視化したかった
ライブラリの開発者が内部の仕組みをよく知っているので、その人に質問することは自然な流れだと思います。
ただ、開発者が休みだったり、ミーティングだったりで質問をいただいても、すぐに答えることができない可能性があります。
その際に、コントリビューターの表示があって、それが同じチームの人であれば、ライブラリのコードを触っていただいているので、解決策を持っている可能性があります。
このように、ライブラリ内の開発に携わったことがある人を表示することで、質問をする先の選択肢が増え、よりライブラリが発展していくと考えています。
そのために、プロジェクトにさえアクセスすれば、誰が開発に携わったことがあるかを可視化したいと思いました。
以上が動機です。
では、実際のコードの説明をしていきます。
## マージリクエストがマージされた時に、READMEにマージリクエストを出した人を追加する
### 全体のコード
.gitlab-ci.yamlに以下の記載をします。
```yaml
stages:
  - update_readme
update_readme_job:
  stage: update_readme
  image: alpine:latest
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: always
    - when: never
  # このジョブはマスターブランチへのマージ時のみ実行
  script:
    - apk add --no-cache git curl jq
    - git config --local user.name "GitLab CI"
    - git config --local user.email "gitlab-ci@example.com"
    - git checkout $CI_COMMIT_REF_NAME
    - |
      # マージリクエスト情報の取得
      MR_IID=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests?state=merged&order_by=updated_at&sort=desc" | jq '.[0].iid')
      MR_AUTHOR_NAME=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${MR_IID}" | jq -r '.author.username')
      MR_AUTHOR_ID=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${MR_IID}" | jq -r '.author.id')
      MR_AUTHOR_SHOW_NAME=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${MR_IID}" | jq -r '.author.name')
      
      # READMEファイルの更新
      if ! grep -q "<!-- CONTRIBUTORS -->" README.md; then
        echo -e "\n## Contributors\n<!-- CONTRIBUTORS -->\n" >> README.md
      fi
      
      if ! grep -q "alt='${MR_AUTHOR_NAME}'" README.md; then
        sed -i "/<!-- CONTRIBUTORS -->/a [<img src='${CI_SERVER_URL}/uploads/-/system/user/avatar/${MR_AUTHOR_ID}/avatar.png' alt='${MR_AUTHOR_NAME}' width='40'>](@${MR_AUTHOR_NAME})" README.md
              # 変更をコミットしてプッシュ
        git add README.md
        git commit -m "Update README with new contributor: ${MR_AUTHOR_SHOW_NAME}" || echo "No changes to commit"
        git push "https://gitlab-ci-token:${GITLAB_ACESS_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git" HEAD:$CI_COMMIT_REF_NAME
      fi
```
後は、README.mdファイルを用意するだけで実装は完了ですが、GITLAB_ACESS_TOKENを取得し、設定に追加する必要があります。
### アクセストークンを取得・設定する
GitLab左上のアイコンをクリックし、「Preferences」をアクセスします。
![2024-10-14_13h07_11.png](/images/add-contributer-in-gitlab-by-ci/2024-10-14_13h07_11.png)
その後、サイドメニューの「Acess tokens」をクリック、表示される画面の「Add new token」をクリックします。
![2024-10-14_13h08_34.png](/images/add-contributer-in-gitlab-by-ci/2024-10-14_13h08_34.png)
任意のトークン名を設定し、トークンの権限は「read_api」と「write_repository」にチェックをつけて、作成します。
![2024-10-14_13h11_10.png](/images/add-contributer-in-gitlab-by-ci/2024-10-14_13h11_10.png)
そして、このトークンをCIのタイミングで参照できるようにします。
プロジェクトに戻り、サイドメニューの「Settings」から「CI/CD」をクリックします。
![2024-10-14_12h58_01.png](/images/add-contributer-in-gitlab-by-ci/2024-10-14_12h58_01.png)
表示される画面にある、「Variables」を開き「Add variable」をクリックします。
![2024-10-14_12h58_51.png](/images/add-contributer-in-gitlab-by-ci/2024-10-14_12h58_51.png)
すると、右に以下のサイドメニューが表示されます。
![2024-10-14_13h00_33.png](/images/add-contributer-in-gitlab-by-ci/2024-10-14_13h00_33.png)
このトークンは外に漏れていい値ではないので、「Masked and hidden」を選択し、画面上やCI上で表示させないようにします。
また、Protect variableにもチェックをつけて、特定のブランチからしか参照できないようにします。
後は、任意の変数名を設定し(今回はGITLAB_ACESS_TOKENという名前にしています)、先程取得したトークンを値にセットします。
これで、準備は完了です。
## 動作確認
任意のブランチを切り、以下の差分を持つマージリクエストを作成します。
![2024-10-14_13h23_50.png](/images/add-contributer-in-gitlab-by-ci/2024-10-14_13h23_50.png)
README.mdファイルを作成するマージリクエストになります。
これをmainブランチにマージします。
パイプラインが完了すると、README.mdは以下のようにアイコン画像が追加されています。
![2024-10-14_13h25_42.png](/images/add-contributer-in-gitlab-by-ci/2024-10-14_13h25_42.png)
さらにアイコンをホバーすると、アイコン画像と名前が表示されます。(名前は黒塗り部分です)
![2024-10-14_13h26_39.png](/images/add-contributer-in-gitlab-by-ci/2024-10-14_13h26_39.png)
これで、マージリクエストを出してくれた人がコントリビューターとして、README.mdに追加されました。
これで、プロジェクトに対してマージリクエストを出してくれた人を表示することで、わずかではありますが感謝を示せるようになったかと思います。
## コードの解説
ここからは、.gitlab-ci.yamlが何をしているかについて解説します。
ただし、筆者はCIの専門家ではなく、何なら素人よりなので、誤り・不足の可能性は大いにあります。
その点はご留意いただきたいのと、あわよくばコメントなどでご指摘いただけますと幸いです。
### stageの定義
以下のように、CIで実行する名前を決め、それをstages配下に列挙します。
```yaml
stages:
  - update_readme
update_readme_job:
  stage: update_readme
```
stageは[ドキュメント](https://gitlab-docs.creationline.com/ee/ci/yaml/#stages)を確認すると、以下の機能になっています。
> ジョブのグループを含むステージを定義するには、`stages` を使用します。ジョブを特定のステージで実行するように設定するには、ジョブで[`stage`](https://gitlab-docs.creationline.com/ee/ci/yaml/#stage) を使用します。
> 

「update_readme_job」というジョブを、特定のタイミングで実行するために行う設定となっています。
正直、今回は単一のジョブしか定義していないので、この設定は無くても動くと思います。
しかし、実際の開発ではもっと他のジョブも実行されるので、その際に調整が効きやすくするため、記載しています。
詳しくは、ドキュメントに書いてありますが、同じstageを設定したjobは並列で実行され、前のステージが終わった後に、次のステージに属するjobを実行するなど色々と使い勝手は良いです。
### Linuxイメージの取得
今回はjobの中でコマンド実行を行うことで、README.mdにコントリビューターを追加できるようにしています。
なので、コマンドを実行するためのLinux環境を取得します。
それが以下のimage部分になります。
```yaml
update_readme_job:
  # ...略
  image: alpine:latest
```
今回は[Alpine](https://www.alpinelinux.org/)を使用しています。
Alpineの特徴の一つとして、軽量であることが挙げられます。
今回は、アプリケーションを動かすわけではなく、コマンドを実行さえできれば良いので、なるべく早くCIが終わるようにするために、Alpineを選択しました。
ただし、以下記事のようにやりたいコマンドによっては、Alpineでは不都合な場合があります。
https://engineering.nifty.co.jp/blog/26586
今回は、gitコマンド・curlコマンド・jqコマンドの単純機能しか使用しないので、Alpineでいいかなと思います。
これで、linux環境ができました。
### ジョブが動くタイミングを指定
ジョブが実行されるタイミングはrulesで定義します。
```yaml
update_readme_job:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: always
    - when: never
```
最初のruleで、mainブランチにコミットされた際にjobを実行すると定めています。
このCI設定したプロジェクトでは、「マージリクエストがマージされる＝mainブランチにコミットする」となっているので、上記の記載をしています。
それ以外の操作については、`when: never`とすることでjobが動かないようにします。
### スクリプトの実行
ここが、実際にREADME.mdにコントリビューターの追記を行う部分となっています。
```yaml
script:
		# コマンド処理に必要なパッケージを取得 (--no-cacheにしているが、別にキャッシュありでもいい気はしています)
    - apk add --no-cache git curl jq
    - git config --local user.name "GitLab CI"
    - git config --local user.email "gitlab-ci@example.com"
    - git checkout $CI_COMMIT_REF_NAME
    - |
      # マージリクエスト情報の取得
      MR_IID=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests?state=merged&order_by=updated_at&sort=desc" | jq '.[0].iid')
      MR_AUTHOR_NAME=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${MR_IID}" | jq -r '.author.username')
      MR_AUTHOR_ID=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${MR_IID}" | jq -r '.author.id')
      MR_AUTHOR_SHOW_NAME=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${MR_IID}" | jq -r '.author.name')
      
      # READMEファイルの更新
      if ! grep -q "<!-- CONTRIBUTORS -->" README.md; then
        echo -e "\n## Contributors\n<!-- CONTRIBUTORS -->\n" >> README.md
      fi
      
      if ! grep -q "alt='${MR_AUTHOR_NAME}'" README.md; then
        sed -i "/<!-- CONTRIBUTORS -->/a [<img src='${CI_SERVER_URL}/uploads/-/system/user/avatar/${MR_AUTHOR_ID}/avatar.png' alt='${MR_AUTHOR_NAME}' width='40'>](@${MR_AUTHOR_NAME})" README.md
              # 変更をコミットしてプッシュ
        git add README.md
        git commit -m "Update README with new contributor: ${MR_AUTHOR_SHOW_NAME}" || echo "No changes to commit"
        git push "https://gitlab-ci-token:${GITLAB_ACESS_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git" HEAD:$CI_COMMIT_REF_NAME
      fi
```
まずは以下の部分で、必要なパッケージとコミット・プッシュするための一時的なgitユーザーを設定します。
また、chekoutで$CI_COMMIT_REF_NAMEのブランチに移動します。
```yaml
    - apk add --no-cache git curl jq
    - git config --local user.name "GitLab CI"
    - git config --local user.email "gitlab-ci@example.com"
    - git checkout $CI_COMMIT_REF_NAME
```
今回はmainブランチにコミットされたタイミングでjobが実行されるので、$CI_COMMIT_REF_NAMEはmainブランチを指します。
その後、GitLabが提供しているAPIから値を取得し、欲しい値をjqコマンドで抽出しています。
```yaml
# マージリクエスト情報の取得
MR_IID=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests?state=merged&order_by=updated_at&sort=desc" | jq '.[0].iid')
MR_AUTHOR_NAME=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${MR_IID}" | jq -r '.author.username')
MR_AUTHOR_ID=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${MR_IID}" | jq -r '.author.id')
MR_AUTHOR_SHOW_NAME=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${MR_IID}" | jq -r '.author.name')
      
```
それぞれのスクリプトは以下のことを行っています。
1. 更新日時の降順でマージリクエストの情報を取得し、最初の値に含まれるマージリクエストのIDを変数に定義しています。
2. 1で取得したマージリクエストIDを元に、マージリクエストのAuthorのアカウント名・アカウントID・名前を取得し、変数に定義しています。

それぞれ、トークンが必要となるので、先程設定したトークンを参照するようにしています。
これで、README.mdに記載するために必要な情報が揃いました。
残りで実際にファイルへ記載をし、変更を反映させます。
```yaml
    	# READMEファイルの更新
      if ! grep -q "<!-- CONTRIBUTORS -->" README.md; then
        echo -e "\n## Contributors\n<!-- CONTRIBUTORS -->\n" >> README.md
      fi
      
      if ! grep -q "alt='${MR_AUTHOR_NAME}'" README.md; then
        sed -i "/<!-- CONTRIBUTORS -->/a [<img src='${CI_SERVER_URL}/uploads/-/system/user/avatar/${MR_AUTHOR_ID}/avatar.png' alt='${MR_AUTHOR_NAME}' width='40'>](@${MR_AUTHOR_NAME})" README.md
              # 変更をコミットしてプッシュ
        git add README.md
        git commit -m "Update README with new contributor: ${MR_AUTHOR_SHOW_NAME}" || echo "No changes to commit"
        git push "https://gitlab-ci-token:${GITLAB_ACESS_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git" HEAD:$CI_COMMIT_REF_NAME
      fi
```
まず、grepコマンドで、README.mdに「`<!-- CONTRIBUTORS -->`」がないかを探しています。
対象の文字列が無ければ、「`\n## Contributors\n<!-- CONTRIBUTORS -->\n`」をREADME.mdへ追記するようにしています。
これによって、コントリビューターを追加するための設定がREADME.mdに存在しないことを防ぎます。
まあ、一回しか動かないので、正直なくてもいいです。
次のif文で、「`alt=’マージリクエストを作成した人のユーザー名’`」という文字列がREADME.mdにないかをgrepコマンドで検索しています。
もし無ければ、sedコマンドで「`<!-- CONTRIBUTORS -->`」の文字列のすぐ後に、「`[<img src='${CI_SERVER_URL}/uploads/-/system/user/avatar/${MR_AUTHOR_ID}/avatar.png' alt='${MR_AUTHOR_NAME}' width='40'>](@${MR_AUTHOR_NAME})`」を追記するようにします。
imgタグを用いて記載することで、README.mdに任意の大きさのアイコン画像を表示させることができます。
そして、[表示される内容](@ユーザー名)とすることで、ホバーした時に人の情報を表示することができます。
アカウント画像は、「使用しているGitLabのルートURL/uploads/-/system/user/avatar/ユーザーID/avatar.png」で取得できので、imgタグのsrc属性にその値を設定しています。
後は、この変更をadd・commit・pushすることで、プロジェクトに反映させています。
なお、pushの際に使用しているURLは[こちらのドキュメント](https://gitlab-docs.creationline.com/ee/ci/jobs/ci_job_token.html)を元に設定しています。
今回はトークンをJOB_TOKENではなく、プロジェクトに書き込み権限を持つトークンを設定することで、pushを可能としています。
以上で、README.mdにマージリクエストがマージされたタイミングで、Authorの情報を追加することができるようになりました。
GitHubで自分のプルリクエストがマージされ、コントリビューターとして記載されたときは、嬉しさを感じました。
そのうれしさを、今回搭載した機能で少しでも感じてもらえたらと思っています。
## おわりに
今回はGitLabのCIを用いて、マージリクエストの作成者をREADME.mdに追記できるようにしました。
これによって、疑似的なコントリビューター機能を搭載することができました。
内容としては、小ネタにはなりますが、こういった小ネタがより活発なマージリクエストに繋がればいいなと思います。
ここまで読んでいただきありがとうございました。