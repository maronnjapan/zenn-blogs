---
title: "GitLabでREADMEに疑似的なコントリビューター機能を設ける(改)"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitLab","CI"]
published: true
---
## はじめに
[前回の記事](https://zenn.dev/maronn/articles/add-contributer-in-gitlab-by-ci)で、GitLabで疑似的なコントリビューター機能を搭載する方法について紹介しました。
上記によって、マージリクエストをマージした際に、マージリクエストを出してくれた人のアイコンをREADMEに記載することができます。
ただし、[以前の記事](https://zenn.dev/maronn/articles/add-contributer-in-gitlab-by-ci)は以下の改善点もありました。
- メンテナーのマージリクエストははじくなどの設定がなく、全ての人がコントリビューターとして記載されてしまう。
- mainブランチへの直コミットでも、コントリビューター搭載機能が動いてしまう。
- アイコンを設定していないアカウントの場合、デフォルトアイコンが表示されない
- READMEにアイコンの有無で判定していたので、貢献度の比較ができない

今回の記事は、それら問題を改善したものになります。
良ければ読んでください。
## 全体のコード
まずは、全体のコードを展開します。
今回新規で追加した処理はコメントの先頭に「New：」と記載しています。
なので、今回搭載した機能のみ気になる方は、その部分のみ参照してください。
```yaml
stages:
  - update_readme
update_readme_job:
  stage: update_readme
  image: alpine:latest
  rules:
    # mainブランチにコミットがあったら実行
    - if: $CI_COMMIT_BRANCH == "main"
      when: always
  script:
    - apk add --no-cache git curl jq
    - git config --local user.name "GitLab CI"
    - git config --local user.email "gitlab-ci@example.com"
    - git checkout $CI_COMMIT_REF_NAME
    - |
			# New：コミットした人のアカウントを取得
      COMMITTER_LOGIN="${GITLAB_USER_LOGIN}"
      # New：除外アカウントの変数が設定されているかチェック
      if [ -z "${EXCLUDED_ACCOUNTS}" ]; then
        echo "Warning: 変数EXCLUDED_ACCOUNTSが設定されていません。現状は全ての人がコントリビューターの対象となります。"
      else
        # New：カンマで区切られた文字列内にユーザー名が含まれているかチェック
        if echo "$EXCLUDED_ACCOUNTS" | grep -q "$COMMITTER_LOGIN"; then
					# New：存在する場合はメッセージを出して処理を終了する
          echo "Committer ${COMMITTER_LOGIN} はコントリビューターの対象外なため処理を終了します。"
          exit 0
        fi
      fi
    
      # New：マージリクエスト経由かどうかをチェック
			# New：コミットハッシュ値をクエリに設定し、検索をしている
      MR_COUNT=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests?state=merged&target_branch=main&sha=$CI_COMMIT_SHA" | jq length)
    
      # New：直接プッシュの場合は処理を終了
      if [ "$MR_COUNT" -eq 0 ]; then
        echo "マージリクエスト経由ではないコミットのため処理を終了します。"
        exit 0
      fi
      
      # マージリクエスト情報の取得
      MR_IID=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests?state=merged&order_by=updated_at&sort=desc" | jq '.[0].iid')
      MR_AUTHOR_NAME=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${MR_IID}" | jq -r '.author.username')
      MR_AUTHOR_ID=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${MR_IID}" | jq -r '.author.id')
      MR_AUTHOR_SHOW_NAME=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${MR_IID}" | jq -r '.author.name')
      
      
      # READMEファイルの更新
      if ! grep -q "<!-- CONTRIBUTORS -->" README.md; then
        echo -e "\n## Contributors\n<!-- CONTRIBUTORS -->\n" >> README.md
      fi
      # New：アバター画像のURLをチェック
      CUSTOM_AVATAR_URL="${CI_SERVER_URL}/uploads/-/system/user/avatar/${MR_AUTHOR_ID}/avatar.png"
			# New：curlコマンドでレスポンス内容からステータスコードだけを抽出している
			# New：-sはcurlコマンドの進捗の表示を無効化し、-o /dev/nullはレスポンスを非表示にする
			# New：-wはレスポンスから任意の値を抽出する。今回はHTTPステータスコードを取得している
			# New：-wに設定できる値は、例えばこちら(https://shuzo-kino.hateblo.jp/entry/2017/01/08/235111)のページを参照すること
      HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CUSTOM_AVATAR_URL}")
    
      # New：プロジェクト内のデフォルトアバター画像のパス
      # 例: dfault.png にある場合
      DEFAULT_AVATAR_PATH="default.png"
    
      # New：アバター画像が存在しない場合はプロジェクト内のデフォルト画像を使用
      if [ "$HTTP_CODE" != "200" ]; then
        echo "Custom avatar not found, using project default avatar"
        AVATAR_URL="${CI_PROJECT_URL}/-/raw/${CI_COMMIT_REF_NAME}/${DEFAULT_AVATAR_PATH}"
      else
        AVATAR_URL="${CUSTOM_AVATAR_URL}"
      fi
      
      if ! grep -q "alt='${MR_AUTHOR_NAME}'" README.md; then
        sed -i "/<!-- CONTRIBUTORS -->/a [<img src='${CI_SERVER_URL}/uploads/-/system/user/avatar/${MR_AUTHOR_ID}/avatar.png' alt='${MR_AUTHOR_NAME}' width='10'>](@${MR_AUTHOR_NAME})" README.md
        # 変更をコミットしてプッシュ
        git add README.md
        git commit -m "Update README with contributor: ${MR_AUTHOR_SHOW_NAME}" || echo "No changes to commit"
        git push "https://gitlab-ci-token:${GITLAB_ACESS_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git" HEAD:$CI_COMMIT_REF_NAME
      else
        # New：既存ユーザーの場合、現在のwidth値を取得して5を加算
        current_width=$(grep -o "width='\([0-9]\+\)'" README.md | grep -o "[0-9]\+")
        new_width=$((current_width + 5))
        # New：画像URLとwidth値を同時に更新
	      sed -i "s|src='[^']*' alt='${MR_AUTHOR_NAME}' width='[0-9]\+'|src='${AVATAR_URL}' alt='${MR_AUTHOR_NAME}' width='${new_width}'|" README.md
				# 変更をコミットしてプッシュ
        git add README.md
        git commit -m "Update README with contributor: ${MR_AUTHOR_SHOW_NAME}" || echo "No changes to commit"
        git push "https://gitlab-ci-token:${GITLAB_ACESS_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git" HEAD:$CI_COMMIT_REF_NAME
  
      fi
```
以降は追加した内容について、軽く紹介していきます。
ただし、ほぼコメントに書いてあることと同じなので、その点はご留意ください。
## 追加機能①：特定のアカウントを除外
### 対象のコード
```yaml
			# New：コミットした人のアカウントを取得
      COMMITTER_LOGIN="${GITLAB_USER_LOGIN}"
      # New：除外アカウントの変数が設定されているかチェック
      if [ -z "${EXCLUDED_ACCOUNTS}" ]; then
        echo "Warning: 変数EXCLUDED_ACCOUNTSが設定されていません。現状は全ての人がコントリビューターの対象となります。"
      else
        # New：カンマで区切られた文字列内にユーザー名が含まれているかチェック
        if echo "$EXCLUDED_ACCOUNTS" | grep -q "$COMMITTER_LOGIN"; then
					# New：存在する場合はメッセージを出して処理を終了する
          echo "Committer ${COMMITTER_LOGIN} はコントリビューターの対象外なため処理を終了します。"
          exit 0
        fi
      fi
```
### 概要
CIのJobを動かすきっかけになったコミットのアカウントを`${GITLAB_USER_LOGIN}`で取得し、それを変数に設定しています。
後は、GitLab CI/CDのVariablesで設定した変数EXCLUDED_ACCOUNTS内の文字列に、コミットした人のアカウントが含まれているかをチェックしています。
存在している場合、コントリビューターとして記載するアカウントの対象外なため、処理を`exit 0`で終了しています。
## 追加機能②：mainブランチへの直接コミットの除外
### 対象のコード
```yaml
      # New：マージリクエスト経由かどうかをチェック
			# New：コミットハッシュ値をクエリに設定し、検索をしている
      MR_COUNT=$(curl -s -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests?state=merged&target_branch=main&sha=$CI_COMMIT_SHA" | jq length)
    
      # New：直接プッシュの場合は処理を終了
      if [ "$MR_COUNT" -eq 0 ]; then
        echo "マージリクエスト経由ではないコミットのため処理を終了します。"
        exit 0
      fi
```
### 概要
GitLab CIではJobが動くきっかけになったコミットのIDを$CI_COMMIT_SHAで取得できます。
そして、GitLab APIにはマージリクエストを検索するAPIがあります。
これら二つから、検索条件に$CI_COMMIT_SHAのコミット履歴を持つマージリクエストを検索することができます。
$CI_COMMIT_SHAの値を持つマージリクエストがない場合、他の方法でコミットしていることになるので、コントリビューターとして反映しないようにしています。
通常のOSSライブラリでも、マージリクエスト or プルリクエストを出すと思うので、それに合わせました。
## 追加機能③：デフォルトアイコンの設定
### 対象のコード
```yaml
      # New：アバター画像のURLをチェック
      CUSTOM_AVATAR_URL="${CI_SERVER_URL}/uploads/-/system/user/avatar/${MR_AUTHOR_ID}/avatar.png"
			# New：curlコマンドでレスポンス内容からステータスコードだけを抽出している
			# New：-sはcurlコマンドの進捗の表示を無効化し、-o /dev/nullはレスポンスを非表示にする
			# New：-wはレスポンスから任意の値を抽出する。今回はHTTPステータスコードを取得している
			# New：-wに設定できる値は、例えばこちら(https://shuzo-kino.hateblo.jp/entry/2017/01/08/235111)のページを参照すること
      HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -H "PRIVATE-TOKEN: ${GITLAB_ACESS_TOKEN}" "${CUSTOM_AVATAR_URL}")
    
      # New：プロジェクト内のデフォルトアバター画像のパス
      # 例: dfault.png にある場合
      DEFAULT_AVATAR_PATH="default.png"
    
      # New：アバター画像が存在しない場合はプロジェクト内のデフォルト画像を使用
      if [ "$HTTP_CODE" != "200" ]; then
        echo "Custom avatar not found, using project default avatar"
        AVATAR_URL="${CI_PROJECT_URL}/-/raw/${CI_COMMIT_REF_NAME}/${DEFAULT_AVATAR_PATH}"
      else
        AVATAR_URL="${CUSTOM_AVATAR_URL}"
      fi
```
### 概要
[前回の記事](https://zenn.dev/maronn/articles/add-contributer-in-gitlab-by-ci)と同様のURLでユーザーのアイコン画像リンクを定義しています。
しかし、アイコンを設定していない人はそのURLにアイコン画像は存在していません。
その場合は、AVATAR_URLを`AVATAR_URL="${CI_PROJECT_URL}/-/raw/${CI_COMMIT_REF_NAME}/${DEFAULT_AVATAR_PATH}"`でプロジェクトにアップロードしている画像のパスを指定します。
アバター画像があるか否かは、最初に定義したURLをcurlコマンドでリクエストして確認しています。
curlコマンドでのレスポンスから、-wオプションでHTTPステータスコードだけ抽出し、ステータスコードが200以外、すなわち画像が存在しない場合のみ、デフォルトアイコンを定義しています。
## 追加機能④：貢献回数に合わせたアイコン画像の拡大
### 対象のコード
```yaml
# New：既存ユーザーの場合、現在のwidth値を取得して5を加算
current_width=$(grep -o "width='\([0-9]\+\)'" README.md | grep -o "[0-9]\+")
new_width=$((current_width + 5))
# New：画像URLとwidth値を同時に更新
sed -i "s|src='[^']*' alt='${MR_AUTHOR_NAME}' width='[0-9]\+'|src='${AVATAR_URL}' alt='${MR_AUTHOR_NAME}' width='${new_width}'|" README.md
```
### 概要
元々はalt属性にあるアカウント名だけgrepしていましたが、width属性もgrepして取得するようにしました。
そして、取得した値に5を足し、sedコマンドで新しいwidthを設定しています。
なお、この時に先程紹介したアカウントのURLも設定するようにしています。
これは、アイコン画像を設定した時にその変更が反映されるようにしたいためです。
以上機能の紹介でした。
やりたいことは一通りできたので、後は実際に導入していこうと思います。
## おわりに
今回は以前作成したGitLab CIで疑似的なコントリビューターを設定する機能を、改修した記事でした。
要件的にはメインではないにしろ、必要な機能たちだったので、導入するために超えないいけない壁は突破した気がします。
後は実際に運用してまた見えてくる課題もあると思うので、プロジェクトに導入していこうと思います。
ここまで読んでいただきありがとうございました。