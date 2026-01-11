---
title: ""
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
# Auth0のEvent StreamとAWSのEventBridgeを連携して、リトライ可能かつ非同期に情報を連携する

## 1. はじめに

### 1.1 この記事で実現すること
- Auth0でユーザーをブロックしたら、自作アプリ側にもブロック状態が反映される
- デモ動画を埋め込み（管理画面でブロック → Auth0イベント発火 → アプリ側でブロック状態反映の流れ）

### 1.2 この記事のアプローチ
- Auth0のEvent StreamとAWS EventBridgeを使い、非同期かつリトライ可能な構成で実現する
- 前回記事（Webhook版）との違い：今回はAWSのマネージドサービスを活用

### 1.3 前回記事との関係
- 前回記事へのリンク（https://zenn.dev/maronn/articles/notify-block-user-by-auth0-event-stream）
- 前回はWebhookで直接受け取る構成だった
- 今回はEventBridge経由でより堅牢な構成にする

## 2. 全体構成

### 2.1 構成図
- Draw.ioの構成図を埋め込み

### 2.2 処理フローの説明
- Auth0：ユーザーブロックイベントが発生
- EventBridge：Auth0からのイベントを受信（パートナーイベントソース経由）
- EventBus：Auth0のイベントのみをフィルタリングしてSQSへ転送
- SQS：イベントをキューイング、失敗時はDLQに退避
- Lambda：キューからイベントを取得し、デモアプリのAPIを呼び出す
- Demo App（Cloudflare Workers + Vike/Hono）：ブロック状態を反映

### 2.3 各コンポーネントの役割
- EventBridge + EventBus：イベントの受信とルーティング
- SQS：非同期処理の実現、バッファリング
- DLQ：失敗イベントの退避と再処理の起点
- Lambda（Node.js）：イベント処理とデモアプリへの通知

## 3. Event Streamとは

### 3.1 概要
- Auth0が提供するイベント配信機能
- OrganizationとUserに関するイベントを検知・配信する

### 3.2 配信先の選択肢
- Webhook URL
- AWS EventBridge
- 今回はEventBridgeを使用

### 3.3 参考ドキュメント
- 公式ドキュメントへのリンク（https://auth0.com/docs/customize/events）

## 4. 今回の構成の利点

### 4.1 非同期処理によるブロッキングの回避
- イベント発生元（Auth0）の処理をブロックしない
- Webhookでも裏で非同期処理は可能だが、別途キューの仕組みが必要
- サーバーレス環境（例：Cloudflare Workers）でWebhookを受けると、EventEmitterなどでイベントが消失するリスクがある

### 4.2 リトライ機構による一時的障害への耐性
- SQSの標準的なリトライ機能を利用できる
- Webhookだと失敗時に処理が消失する
- Webhookで内部リトライを実装しても、連携先の長時間障害には対応しづらい

### 4.3 DLQによる失敗イベントの再処理
- 失敗したイベントをDLQに退避
- 連携先の復旧後に再処理が可能
- 手動または自動で再投入できる

## 5. 今回の構成の注意点

### 5.1 イベント順序の担保の難しさ
- 今回はFIFOキューを使っていないため、順序は保証されない
- FIFOキューを使っても、DLQとメインキューを合わせた順序担保は困難
- 例：DLQに入ったイベントが再処理されると、後続の正常イベントより後に処理される可能性

### 5.2 冪等性の確保が必要
- 同じイベントが複数回処理される可能性がある
- 今回のデモでは冪等性は担保していない
- 本番運用では対策が必要（例：event_idをキーにした重複チェック）

### 5.3 SQS/Lambdaの設定値
- visibility_timeout_seconds：90秒
- maxReceiveCount：3回（3回失敗でDLQへ）
- Lambda側のリトライ：3回
- これらの値に深い検討はしていないが、参考値として記載

### 5.4 コストへの意識
- 詳細なコスト計算は記載しない
- ただし、むやみにイベントを流すのは避けるべき
- 本番導入時はイベント量とコストの見積もりを推奨

## 6. 設定手順

### 6.1 前提条件
- Cloudflareアカウントがあること
- Auth0テナントが作成済みであること
- AWSアカウントがあり、IAM Identity CenterでAdmin権限を持つSSOアカウントがあること

### 6.2 サンプルリポジトリ
- GitHubリポジトリへのリンク
- READMEに詳細な手順を記載している
- このブログでは手順の詳細は割愛

### 6.3 設定の流れ（概要のみ）
1. AWSのSSOとAuth0のTerraform用アプリケーションをCLI経由で設定
2. Auth0側でEvent StreamをTerraform経由で生成
3. AWS側でパートナーイベントソースの紐づけをCLIで行う
4. Terraform経由でAWSの残りの構成（EventBus、SQS、DLQ、Lambda）を設定
5. デモアプリ用のAuth0アプリ作成とデモアプリのデプロイをCLIで行う

## 7. 個人的にハマったポイント

### 7.1 Auth0のEventBridge設定とAWSの紐づけ
- Auth0のUIではAWSアカウントIDとリージョンしか設定しない
- AWSとどう連携するのか最初は不明だった
- 答え：AWSのパートナーイベントソースにAuth0のイベントソースが作成される
- それをAWS側のイベントバスに関連付ける必要がある
- 参考記事（https://dev.classmethod.jp/articles/auth0-log-streams-to-amazon-eventbridge-with-cloudformation/）

### 7.2 パートナーイベントソースとTerraformの連携
- パートナーイベントソースのイベントバスへの登録はTerraformではできない
- CLIまたはUIで手動設定が必要（サンプルではCLIスクリプトで対応）
- Terraformで他リソースを紐づける方法：`aws_cloudwatch_event_bus`データソースでイベントバスを取得し、それに対してルールやSQSを紐づける
- 注意：dataソースなので`terraform destroy`してもパートナーイベントソースの登録は解除されない

### 7.3 詳細な解消方法
- サンプルリポジトリのスクリプトを参照（該当ファイルへのリンク）

## 8. WebhookとEventBridge、どちらを選ぶか

### 8.1 基本的な推奨
- コストが許すならEventBridge（AWS連携）を推奨

### 8.2 EventBridgeを推奨する理由
- Webhookはトークン管理などセキュリティ面を自前で担保する必要がある
- EventBridgeならAWSのマネージドな仕組みでセキュリティリスクを軽減できる
- リトライ設計がしやすく、SQSやDLQとの組み合わせで拡張性がある

### 8.3 Webhookを選ぶケース
- AWSを使っていない環境
- コストを最小限に抑えたい場合

## 9. 終わりに

### 9.1 まとめ
- Auth0のEvent StreamとAWS EventBridgeを連携し、非同期かつリトライ可能な構成を実現した
- Webhookと比較して、セキュリティとリトライ設計の面でメリットがある

### 9.2 今後の発展
- 冪等性の担保
- 他のイベント種別への拡張
- モニタリング・アラートの追加

### 9.3 参考リンク
- サンプルリポジトリ
- Auth0 Event Stream公式ドキュメント
- 前回記事（Webhook版）