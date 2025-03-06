---
title: "OpenID ConnectでのOpenID ProviderをHonoとNext.jsで作成する ①アプリケーションの準備編"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Hono","NextJS","OpenIDConnect"]
published: true
---
## はじめに
勉強のため、定期的にOpenID Connectを構築しています。
まだまだあまりにも不足があるのですが、とりあえず一連の流れぽいものは実装できたので今回記事にしています。
なお、上記にもある通り実装には不足分がかなりあります。
そのため、この記事は個人のメモ要素が強く、正確性に欠く記載が散見されることはご留意ください。
それでは、始めます。
## 今回の構成
今回は以下のように実装しています。
### アプリケーション
![image.png](/images/development-op-part1/image.png)
フロントエンドはNext.jsでAPI部分はHonoにしています。
Hono、もしくはNext.jsだけでいいのでは？と思われた方はいらっしゃると思いますが、その通りです。
ただ、今回はAPIとして完全に分離して、そのAPIは様々な環境から呼び出せるようにしたいと思い二つを使用した構成にしています。
### 実行環境
Next.jsは[Vercel](https://vercel.com/)で動かし、Honoは[Cloudflare Workers](https://www.cloudflare.com/ja-jp/developer-platform/products/workers/)で実行しています。
これについては、サクッとデプロイする方法を最優先したため、各フレームワークに新和性が高いものを選択しました。
Cloudflare Workersについては、加えてエッジコンピューティングで実行される環境での開発を体験したかったのもあります。(Vercelも同じエッジコンピューティングで実行されると思いますが、Vercelに関してはあまり深く考えていないです）
### DB・ストレージ
データベースには[Cloudflare D1](https://www.cloudflare.com/ja-jp/developer-platform/products/d1/)を使用し、Key-Valueストレージは[Cloudflare Workers KV](https://www.cloudflare.com/ja-jp/developer-platform/products/workers-kv/)を使用しています。
MySQLやPostgreSQLでないといけない要件がいまのところなく、扱うテーブル構造も複雑ではないのでCloudflare Workersでの使用しやすさを優先しました。
なお、今回の実装でDB操作のORMにPrismaを使用しているのですが、[ドキュメント](https://www.prisma.io/docs/orm/overview/databases/cloudflare-d1#limitations)で記載があるようにCloudflare D1を使用する場合、トランザクションは使用できません。
そのため、業務で使用する際にはトランザクションが無くても問題ないかを確かめた上で、使用する必要があるのはご留意ください。
Key-Valueストレージの選択についても、Cloudflare Workersとの新和性を重視しました。
ただし、[ドキュメント](https://www.notion.so/OpenID-Connect-OpenID-Provider-Hono-Next-js-1a427889392e80d9b8aed8204cd1d37d?pvs=21)にはネットワークによって60秒ほど反映へ遅延が起きるという記載があるので、本当に要件を満たすことができるかは検討する必要があります。
それでもやっぱり、Cloudflare系のものを使用すると構築がとても楽だったので、その誘惑には抗うことが出来ませんでした。
以上がざっくりとした構成になっています。
以降は実際に準備していきます。
## アプリケーションの準備
### 各種アプリのセットアップ
ここではアプリケーションをローカルでまず構築しています。
なお、以降の話はnpmの使用を前提にしています。
他のツールの方がなじみがある場合は、適宜置き換えてください。
まずは、Honoの準備をします。
[ドキュメント](https://hono.dev/docs/)を基に、`npm create hono@latest`を実行します。
すると、色々と質問が表示されるので、適宜入力します。
ただし、以下の実行環境についての質問はCloudflare Workersを選択しています。
![2025-02-24_10h08_47.png](/images/development-op-part1/2025-02-24_10h08_47.png)
後は、プロジェクトがインストールできたらnpm run devでアプリケーションを起動して、「Hello Hono!」が表示されることを確認してください。
以上一旦Honoの準備が完了です。
次にNext.jsですが、こちらも[ドキュメント](https://nextjs.org/docs/app/getting-started/installation)通りです。
Next.jsの質問については、以下のように回答しています。
![2025-02-24_10h22_01.png](/images/development-op-part1/2025-02-24_10h22_01.png)
Next.jsの大まかなセットアップは以上ですが、一点追加の作業がありpacke.jsonのscriptsに以下の記載を行います。
```tsx
"dev:https": "next dev --turbopack --experimental-https"
```
このコマンドはhttpsでアプリケーションを起動するために設定しています。([ドキュメント](https://www.notion.so/Next-js-tRPC-Auth0-Markdown-f182e3d4ac5244428e51972b0b718438?pvs=21)参照)
httpsで実行するのは、Relying Partyは[@auth0/auth0-react](https://github.com/auth0/auth0-react)を使用してOpenID Providerとやり取りするためです。
@auth0/auth0-reactは@auth0/auth0-spa-jsをReact用にラップしたものですが、@auth0/auth0-spa-jsは[こちらのコード](https://github.com/auth0/auth0-spa-js/blob/main/src/utils.ts#L221)で書かれているように、httpsで動いているOpenID Providerのみへリクエストを送る想定でいます。
今回Relyting Partyからリクエストを受け取るのは、まずこのNext.jsのためここをhttpsすることでリクエストができるようにします。
以上を達成するために、--experimental-httpsオプションを設定したスクリプトを追加します。
これらでアプリケーションのセットアップは完了しました。
### HonoでAPI定義書を作成する
各種アプリのセットアップはできたので、今度はNext.jsからHonoのアプリを呼べるようにします。
まずは、Hono側でOpenAPIを使用したAPI定義書の提供を行います。
といっても、とても簡単でsrc直下のindex.tsへ以下のように実装するだけです。
```tsx
import { OpenAPIHono } from '@hono/zod-openapi'
const app = new OpenAPIHono()
app.doc('/doc', {
  openapi: '3.0.0',
  info: {
    title: 'OpenID ConnectのAPI',
    version: '1.0.0'
  }
})
export default app
```
こうすることで、定義したAPIをJSON形式で提供することができます。
では、実際にAPIを作ってみます。
流れとしては、以下の通りです。
1. リクエストとレスポンスのスキーマを`@hono/zod-openapi`提供のzオブジェクトを使用し、定義する
2. 1を使いつつ、ルーティングの定義を行う
3. ルーティングを基に、内部の処理を記載する

これに沿って、/oauth/tokenのPOSTエンドポイントを作ってみます。
まず、以下のようなschemaを定義します。
```bash
import { z } from '@hono/zod-openapi'
export const CreateTokenParamSchema = z.object({
    client_id: z.string().optional().openapi({ title: 'クライアントID', description: 'Publicクライアントからのリクエスト時は必須' }),
    redirect_uri: z.string().optional().openapi({ title: 'リダイレクトURI', description: 'Publicクライアントからのリクエスト時は必須' }),
    grant_type: z.enum(['authorization_code', 'refresh_token']).openapi({ title: '認可タイプ', description: 'authorization_code: 認可コード取得, refresh_token: リフレッシュトークン取得' }),
    code: z.string().openapi({ title: '認可コード' }),
    code_verifier: z.string().openapi({ title: '認可コードベリファイア', description: '仕様書では推奨だが、セキュリティを考え必須とする' }),
})
export type CreateTokenParamType = z.infer<typeof CreateTokenParamSchema>
export const CreateTokenResponseSchema = z.object({
    access_token: z.string().openapi({ title: 'アクセストークン' }),
    token_type: z.enum(['Bearer']).openapi({ title: 'トークンタイプ' }),
    expires_in: z.number().openapi({ title: '有効期限' }),
    refresh_token: z.string().optional().openapi({ title: 'リフレッシュトークン', description: 'リフレッシュトークンの要求もあった時のみ返却する' }),
    id_token: z.string().openapi({ title: 'IDトークン' }),
})
```
`@hono/zod-openapi`のzオブジェクトは基本的にZodと書き方が一緒です。
openapiメソッドについては、説明を記載することでAPI定義書にコメントをつけることができます。
また、このスキーマを用いてルーティングを定義することで、リクエストのバリデーションを行ってくれます。
なので、内部でリクエストのバリデーションを書かなくても良くなり漏れが減ります。
型レベルでスキーマを要求するので、必ず定義する必要がある設計となっています。
この制限は個人的に気に入っています。
どうしてもリクエストのバリデーションは後回しにしがちなのですが、後回しせずに取り組むことを強制するためです。
スキーマを定義したら、以下のようなルーティングを作成します。
```bash
import { createRoute } from "@hono/zod-openapi";
import { CreateTokenParamSchema, CreateTokenResponseSchema } from "./schema";
export const createPostRouter = createRoute({
    method: 'post',
    path: 'oauth/token',
    request: {
        body: { content: { "application/json": { schema: CreateTokenParamSchema } } }
    },
    responses: {
        201: {
            content: {
                "application/json": {
                    schema: CreateTokenResponseSchema
                }
            },
            description: 'トークンの返却'
        }
    }
})
```
直感的に分かる内容なので、詳細は省きます。
レスポンスについては、補完が出ないので返したいステータスコードを自分で記載する必要があるのは注意です。
ルーティングもできたので、Honoのアプリケーションに教えこみます。
```tsx
import { createPostRouter } from "./router";
import type app from "../..";
export const registerTokenRoutes = (baseApp: typeof app) => {
    baseApp.openapi(createPostRouter, async (c) => {
       /** 内部の処理 */
        return c.json({ /** レスポンススキーマで定義した値 */ }, 201,)
    })
}
```
openapiとして提供するには、OpenAPIHonoのopenapiメソッドを使用します。
第一引数にルーティングの定義を記載し、第二引数で実際の処理のコールバック関数を実装します。
なお、今回registerTokenRoutes関数の引数の型を`OpenAPIHono`ではなく、src直下のindex.tsが提供しているappにしているのは理由があります。
それは、Bindingを使用した場合でも型エラーを回避するためです。
Honoでは、Cloudflare Workersで動かす時に[Binding](https://hono.dev/docs/getting-started/cloudflare-workers#bindings)という機能を使用できます。
Bindingはざっくり言うと、環境値をアプリケーションのインスタンス時に紐づける機能です。
Bindingはリテラルは環境変数だけでなく、Cloudflare D1やCloudflare Workers KVなどCloudflareの機能もアプリケーションと紐づけることができます。
このBindingの使い方は例えば以下のようになります。
```tsx
type Bindings = {
  DB: D1Database
  "kv-store": KVNamespace,
  ISSUER_URL: string,
}
const app = new OpenAPIHono<{ Bindings: Bindings }>()
```
上記のようにすることで、アプリケーションはenvプロパティを使用して、紐づけた内容を取得することができます。
ただ、Bindingsを紐づけたOpenAPIHonoの型と、元々のOpenAPIHonoの型は内部的に異なっています。
よって、生のOpenAPIHono型を使うとエラーが起きるため、src直下のindex.tsで定義したappをtypeofすることでエラーを回避しています。
以上で教え込む準備ができたので、src直下のindex.tsを変更します。
```tsx
import { OpenAPIHono } from '@hono/zod-openapi'
const app = new OpenAPIHono()
registerTokenRoutes(app)
app.doc('/doc', {
  openapi: '3.0.0',
  info: {
    title: 'OpenID ConnectのAPI',
    version: '1.0.0'
  }
})
export default app
```
これでHonoを起動し、http://localhost:8787/docにアクセスすると、以下の値が出力されます。
```json
{
	"openapi": "3.0.0",
	"info": {
		"title": "OpenID ConnectのAPI",
		"version": "1.0.0"
	},
	"components": {
		"schemas": {},
		"parameters": {}
	},
	"paths": {
		"/oauth/token": {
			"post": {
				"requestBody": {
					"content": {
						"application/json": {
							"schema": {
								"type": "object",
								"properties": {
									"client_id": {
										"type": "string",
										"description": "Publicクライアントからのリクエスト時は必須",
										"title": "クライアントID"
									},
									"redirect_uri": {
										"type": "string",
										"description": "Publicクライアントからのリクエスト時は必須",
										"title": "リダイレクトURI"
									},
									"grant_type": {
										"type": "string",
										"enum": [
											"authorization_code",
											"refresh_token"
										],
										"description": "authorization_code: 認可コード取得, refresh_token: リフレッシュトークン取得",
										"title": "認可タイプ"
									},
									"code": {
										"type": "string",
										"title": "認可コード"
									},
									"code_verifier": {
										"type": "string",
										"description": "仕様書では推奨だが、セキュリティを考え必須とする",
										"title": "認可コードベリファイア"
									}
								},
								"required": [
									"grant_type",
									"code",
									"code_verifier"
								]
							}
						}
					}
				},
				"responses": {
					"201": {
						"description": "トークンの返却",
						"content": {
							"application/json": {
								"schema": {
									"type": "object",
									"properties": {
										"access_token": {
											"type": "string",
											"title": "アクセストークン"
										},
										"token_type": {
											"type": "string",
											"enum": [
												"Bearer"
											],
											"title": "トークンタイプ"
										},
										"expires_in": {
											"type": "number",
											"title": "有効期限"
										},
										"refresh_token": {
											"type": "string",
											"description": "リフレッシュトークンの要求もあった時のみ返却する",
											"title": "リフレッシュトークン"
										},
										"id_token": {
											"type": "string",
											"title": "IDトークン"
										}
									},
									"required": [
										"access_token",
										"token_type",
										"expires_in",
										"id_token"
									]
								}
							}
						}
					}
				}
			}
		}
	}
}
```
いい感じに定義書を作成してくれましたね。
これで、Hono側でAPIを外部に提供できるようになりました。
### Next.jsでHonoのAPIを呼び出す
次に、Next.jsでHonoのAPIを呼び出すようにします。
ただ呼び出すだけならfetchなり、axiosなりでエンドポイントを書けばいいのです。
とはいえ、補完が出ないのは開発するにおいて結構厳しいので、API定義書をもとにコードを自動生成するようにします。
今回はコード生成のツールとして、[Orval](https://orval.dev/)を使用しますが使い慣れているコード生成ツールで良いと思います。
まず、`npm i -D orval`でOrvalをインストールします。
Orvalはコード生成する時、[デフォルトではAxiosを使用します](https://orval.dev/overview#:~:text=The%20default%20generated%20client%20use%20axios)。
これはfetchなどに変更できますが、特に変更したい動機もないのでデフォルトのままにします。
そのため、`npm i axios`でAxiosもインストールします。
インストールが完了したら、プロジェクト直下にorval.config.tsを作成し、以下のように実装します。
```tsx
import { defineConfig } from 'orval'
export default defineConfig({
  "openid-connect-api-file": {
    input: 'http://localhost:8787/doc',
    output: {
      target: "./src/apis.ts",
      baseUrl: 'http://localhost:8787',
    },
  },
})
```
最初のプロパティは任意の名前で良いです。
そして、プロパティは任意の個数設定ができるので、例えば以下のように記載するとコード生成ファイルが二つできます。
```tsx
export default defineConfig({
  "openid-connect-api-file": {
    input: new URL("/doc", process.env.OPENID_CONNECT_API_URL).href,
    output: {
      target: "./src/apis.ts",
      baseUrl: process.env.OPENID_CONNECT_API_URL,
    },
  },
  "openid-connect-api-file2": {
    input: 'http://localhost:8787/doc',
    output: {
      target: "./src/apis.ts",
      baseUrl: 'http://localhost:8787',
    },
  },
})
```
そのため、複数のOpenAPIの定義書をプロジェクト内で使用することができます。
プロパティ内のinputとoutputについても軽く触れます。
inputは参照するOpenAPI定義書の情報を記載します。
[ドキュメント](https://orval.dev/reference/configuration/input)を見ると、色々と機能はありますが今回は単純にAPI定義書を確認できるURLを記載しています。
outputは出力の仕方を定義します。
こちらは[input以上に色々と設定できます](https://orval.dev/reference/configuration/output)が、今回は出力するファイルパスを定義する`target`と、リクエスト先のURLを決める`baseUrl`のみ使用しています。
以上を定義したら、`npx orval --config ./orval.config.ts`を実行します。
すると、src/apis.tsにapiのコードが自動で生成されます。
後は、アプリケーションで生成された関数をインポートして、実行すればHonoにリクエストが到達します。
なお、クライアントコンポーネントで関数のインポートはできますが、CORSエラーが発生します。
そのため、基本的にはサーバー側でリクエストすることを推奨します。
## おわりに
今回はOpenID Providerを独自実装するためのプロジェクト準備を行いました。
簡単にですが、プロジェクトの準備はできたので次はDBやGitHub Actionsの話をしていければと思います。
引き続きよろしくお願いいたします。