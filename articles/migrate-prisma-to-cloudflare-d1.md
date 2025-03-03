---
title: "Hono上でPrismaのスキーマ変更をCloudflare D1に適用させる"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Cloudflare","CloudflareD1","Prisma"]
published: true 
published_at: 2025-02-27 
---
## はじめに
Honoを使用したアプリケーションをCloudflare Workers上で実行しています。
そして、データベースについてはCloudflare D1を使用しています。
とても開発体験が良く、快適な開発ができています。
ただ、気になる点が全くないわけではありません。
それが、Prismaのスキーマ変更をCloudflare D1に同期させる部分です。
今回は、上記Prismaスキーマ変更をなるべくCloudflare D1にも反映させる流れについてのブログです。
なお、タイトルにHonoと書いてありますが、あまりHono要素はないかもしれませんのでご注意ください。
## この記事のまとめ
[ドキュメント](https://www.prisma.io/docs/orm/overview/databases/cloudflare-d1)をちゃんと読みましょう。
[ドキュメント](https://www.prisma.io/docs/orm/overview/databases/cloudflare-d1)さえ読めば、この記事を読む必要はありません。
## Prisma・Cloudflare D1のマイグレーション
まず、気になる点へ触れる前にPrismaやCloudflare D1のマイグレーション方法を見ていきます。
そして、その後現状ドキュメントで出ている、PrismaのマイグレーションをCloudflare D1へ同期する方法を見ていきます。
### Prismaのマイグレーション
Prismaのマイグレーション方法は[ドキュメント](https://www.prisma.io/docs/orm/prisma-migrate/getting-started)に記載があります。
基本的には以下の通りです。
- schema.prismaを変更する
- `prisma migrate dev --name 変更内容の説明`を実行する
- schema.prismaの変更内容に対応するSQLファイルが作成される

上記を行っていくと、以下のようなディレクトリができます。
![2025-02-24_12h00_44.png](/images/migrate-prisma-to-cloudflare-d1/2025-02-24_12h00_44.png)
これでマイグレーションファイルを作成しつつ、変更をしてくれます。
ただし、このマイグレーションを本番環境やテスト環境に適用する際は、[ドキュメント](https://www.prisma.io/docs/orm/prisma-migrate/workflows/development-and-production#production-and-testing-environments)にある通り`npx prisma migrate deploy`を実行する必要があります。
devコマンドでは、データベース内のデータをリセットすることがあるなど、本番環境で適用するには危険な動きをします。
deployコマンドは、データベースをリセットせずにマイグレーションファイルの内容を適用しようとします。
なので、ローカルでの開発では、devコマンドを使いマイグレーションファイルを作成し、本番環境に適用する時はdeployコマンドで行うという流れになります。
### Cloudflare D1のマイグレーション
Cloudflare D1でのDBのマイグレーションについても[ドキュメント](https://developers.cloudflare.com/d1/reference/migrations/)に記載があります。
CloudflareをCLI操作する時は[Wrangler](https://developers.cloudflare.com/workers/wrangler/)というツールが提供されているので、それを使用します。
このWranglerですが、Honoのプロジェクトを作成する時、実行環境をCloudflare Workersにすれば自動的に設定がされています。
なので、要件的にHonoで問題なければHonoを使うと楽で良いです。
今回は、Honoを使用しているので、Wrangler周りの設定については言及しません。
ただし、wrangler.tomlもしくはwrangler.json(c)に記載されているd1_databasesの部分のコメントアウト外してください。
以下はtomlファイルでの例です。
```yaml
[[d1_databases]]
binding = "MY_DB"
database_name = "my-database"
database_id = "XXXXXX"
```
実際のCloudflare D1を使用して動かすにはちゃんとした値の設定が必要ですが、とりあえずローカルで動かす際にはコメントアウトを外すだけで動くはずです。
DBを使う準備ができたので、マイグレーションファイルを準備します。
ドキュメントにあるようにデフォルトでは、migrationsディレクトリにSQLファイルを記載すれば、その内容を適用するとあります。
なので、基本的には以下のような流れでマイグレーションを行います。
1. migrationsディレクトリを作成します。（ここはなくても良いです。次のコマンドでない場合は、作成するかを聞かれるので。）
2. `wrangler d1 migrations create <DATABASE_NAME> <MIGRATION_NAME>`を実行し、マイグレーション用の空のSQLファイルを作成します。
3. SQL内に適用するSQLを記載します。
4. `wrangler d1 migrations apply <DATABASE_NAME> --local`を実行し、マイグレーションの内容を適用します。(Cloudflare上で作成したD1に適用する場合は、localの代わりにremoteを使用します。)

マイグレーションファイルについては、例えば以下のような形で作られていきます。
![2025-02-24_12h25_53.png](/images/migrate-prisma-to-cloudflare-d1/2025-02-24_12h25_53.png)
## Prismaの内容をCloudflare D1に同期させる
Prisma・Cloudflare D1単体でのマイグレーション方法を確認しました。
ここでは、PrismaのスキーマをCloudfalre D1に同期させる方法を確認します。
といっても、「Cloudflare D1のマイグレーション」でマイグレーションファイルを作ったら以下のコマンドを実行して、applyするだけです。
- 一番最初のマイグレーションファイルに対してschema.prismaの内容を同期する場合

```bash
npx prisma migrate diff \
  --from-empty \
  --to-schema-datamodel ./prisma/schema.prisma \
  --script \
  --output migrations/最初のマイグレーションファイル名.sql
```
- 二つ目以降のマイグレーションファイルに対してschema.prismaの内容を同期する場合

```bash
npx prisma migrate diff \
  --from-local-d1 \
  --to-schema-datamodel ./prisma/schema.prisma \
  --script \
  --output migrations/適用させたいマイグレーションファイル名.sql
```
こうすれば、schema.prismaの内容をCloudflare D1のマイグレーション用SQLファイルに適用してくれます。
後は、「Cloudflare D1のマイグレーション」で記載したのと同様にマイグレーションファイルをapplyするだけです。
## おわりに
今回はPrismaの内容をCloudflare D1のマイグレーションファイルに適用する方法についてみていきました。
Previewとはいえ、Prisma側がCloudflare D1への適用をできるようにしてくれたのは嬉しいです。
そのおかげで簡単に同期ができるようになったので。
ただ、この記事は本来別の内容でした。
元々は、Prismaで作成したマイグレーションファイルをCloudflare D1のマイグレーション用ディレクトリにコピーするという内容でブログを書く想定でした。
その中で調べているうちに、コマンド上で完結することが分かり急遽内容が変更となりました。
正直ドキュメント通りにやればすむ話になってしまったのですが、このまま記事をお蔵入りにするのも悔しかったので投稿いたしました。
ドキュメントはちゃんと読まないといけないなと改めて感じました。
ここまで読んでいただきありがとうございました。
## 参考資料
https://tech.fusic.co.jp/posts/hono-prisma-cloudflare-d1/
https://developers.cloudflare.com/d1/reference/migrations/
https://developers.cloudflare.com/workers/wrangler/
https://www.prisma.io/docs/orm/prisma-migrate/workflows/development-and-production#production-and-testing-environments