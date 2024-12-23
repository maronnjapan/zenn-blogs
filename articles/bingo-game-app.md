---
title: "Next.js、Pusher、Upstashを使ってビンゴアプリを作った話"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Next.js", "Vercel", "Pusher", "Upstash"]
published: true
---
## はじめに
先日忘年会がありました。
何か余興をしたいという話があがり、ビンゴゲームをすることになりました。
そこで、簡単なビンゴアプリを作り、それを使いました。
今回はそのビンゴアプリをどのように実装したかについての記事となります。
## デモ
![bingo-game-demo.gif](/images/bingo-game-app/bingo-game-demo.gif)
上記のように番号を出すための管理番号と、出た番号の一覧を表示する画面を作成しました。
## 実装について
### 使用技術
フレームワーク：Next.js(App Router)
デプロイ先：Vercel
ストレージ：[Upstash For Redis](https://upstash.com/)
通知管理：[Pusher](https://pusher.com/)
フレームワークにNext.jsを選んだのは、バックエンドとフロントエンドの機能を持っていて、私が一番馴染みあるものということで選びました。
後は、Vercelにサクッとデプロイできるというのも理由です。
ストレージについては、Vercelとの新和性を考えて、最初Vercelが提供している[Vercel KV](https://vercel.com/docs/storage/vercel-kv)を使う想定でした。
ただ、ドキュメントに新規の場合は代わりにUpstash KVを使用して欲しい旨の記載があったので、Upstashを選択しました。
なお、探してもUpstash KVという名前は見つかりませんでした。
ただ、Vercel KVはRedisデータベースであることから、Upstashで同じくRedisなのはUpstash For Redisだったので、Upstash For Redisを使用しました。(あっているかはわからないですが、データの保存はできたのでよしとしました。)
通知管理にPusherを使用したのは後述します。
### 構成図
![bingo.drawio.png](/images/bingo-game-app/bingo.drawio.png)
管理画面で出した番号を、Pusherに送信しPusherがユーザー画面に通知しています。
ユーザー画面はPusherとのコネクションを張っているので、通知を受け取ったタイミングで画面へ出すようにしています。
番号の保存は、仮に画面をリロードしてしまったときでも、出た番号の一覧がわかるために行っています。
簡単ではありますが、構成は上記の通りとなります。
### 通知管理に外部サービスを使用した理由
今回の要件的に、複雑な通知はないように感じます。
画面的にも出た番号を伝えるのと、リセット機能くらいです。
なので、わざわざ外部の通知機能を使用する必要はないかと思います。
[サーバー送信イベント](https://developer.mozilla.org/ja/docs/Web/API/Server-sent_events) (Server-Sent Events) や[WebSocket](https://developer.mozilla.org/ja/docs/Web/API/WebSocket)で十分な気がします。
ですが、今回外部の通知機能を使用したのはVercelの都合になります。
Vercelでは、サーバーレス関数をそれぞれ動かすことでアプリケーションを動かします。
そして、各サーバーレス関数は[ドキュメント](https://vercel.com/docs/functions/runtimes#size-limits)を確認すると、実行時間に制限があります。
なので、仮にサーバー側でコネクションを張ったとしても、一定時間が過ぎてしまうと接続が無くなってしまいます。
都度起動するたびに接続を張り直すのも手間ですし、Vercelも以下の記事で言及していますが、サーバーでコネクションを張るのではなく、クライアント側でコネクションすることを選択肢として挙げています。
https://vercel.com/guides/do-vercel-serverless-functions-support-websocket-connections
そのため、今回は上記記事でもリンクがあるPusherを使用し、アプリとしてコネクションを張るのはクライアントとしました。
それによって、リアルタイムでの番号の共有を行いました。
なお、外部サービスとしてPusherを選んだのに深い意図はないです。
最初に検索してみたら、以下の分かりやすい記事があったぐらいの理由です。
https://zenn.dev/hayato94087/articles/0c266563626a27
それでは実際の実装について見ていきます。
なお、実装についてはPusherの通知、Upstash For Redisの保存を中心に記載します。
数値を出す処理については記載しませんので、よろしくお願いします。
## 実装について
### 前提
Vercelのアカウント登録は終わっており、Next.jsのプロジェクトがすでに作成・デプロイされているものとします。
### ストレージの準備
まず、以下のURLで表示される画面で、Upstashをインストールボタンをします。
https://vercel.com/marketplace/upstash
すると、Vercelの紐づけるプロジェクトを選択できるので、作成したプロジェクトを紐づけます。
![2024-12-23_12h03_53.png](/images/bingo-game-app/2024-12-23_12h03_53.png)
紐づけが終わったら、Upstash For Redisのインストールをクリックします。
![2024-12-23_12h05_26.png](/images/bingo-game-app/2024-12-23_12h05_26.png)
その後、メインで使うリージョンを選択するとプランの設定ができるようになるので、無料で使いたい場合はFreeプランを選択します。
すると、データベースの作成と環境変数が表示されるので、プロジェクトの.envに記載します。
プロジェクトとの紐づけもできているかと思いますが、もしプロジェクト内のstorageタブに紐づけが設定されていなければ、同タブ内で接続の設定を行います。
![2024-12-23_12h09_29.png](/images/bingo-game-app/2024-12-23_12h09_29.png)
これで設定は完了です。
### ストレージを操作するための準備
先程も言及したように、Upstash For Redisを作成した時に表示されたシークレット情報を環境変数ファイルに記載します。
そして、`@upstash/redis`をインストールします。
インストール後、任意のファイルで以下の記載をします。
```tsx
import { Redis } from "@upstash/redis";
const redis = new Redis({
    url: process.env.KV_REST_API_URL,
    token: process.env.KV_REST_API_TOKEN,
})
export const db = {
    upsert: async ({ id, data }: { id: string, data: string }) => {
        const result = await redis.get<string | null>(id)
        if (!result) {
            return await redis.set(id, JSON.stringify([data]))
        }
        if (Array.isArray(result)) {
            return await redis.set(id, JSON.stringify([...result, data]))
        }
    }
    ,
    getById: async (id: string): Promise<string[]> => {
        const result = await redis.get<string | null>(id)
        if (!result) {
            return []
        }
        if (Array.isArray(result)) {
            return result as string[];
        }
        throw new Error('不正なデータです')
    },
    delete: async (id: string) => {
        return await redis.del(id)
    }
}
```
今回は数値を配列で扱いたいので、数値で扱うような操作を行っています。
idはURLの/:idを使用する想定でいます。
ルームを超えて値が共有されてしまい、変な挙動になるのを避けたかったためです。
ここまで用意できたら、Route Hadller内で以下のように呼び出しデータの保存や取得を行うようにしました。
```tsx
import { NextRequest, NextResponse } from 'next/server';
import path from 'path';
import { db } from '@/lib/db';
export async function GET(request: NextRequest) {
    const searchParams = request.nextUrl.searchParams;
    const gameId = searchParams.get('gameId');
    if (!gameId) {
        return NextResponse.json({ error: '誤ったurl' }, { status: 400 });
    }
    const fileData = await db.getById(gameId)
    return NextResponse.json({
        fileData
    });
}
export async function POST(request: NextRequest) {
    try {
        const { number, gameId } = await request.json();
       /** 省略 */
        await db.upsert({ data: number.toString(), id: gameId })
        return NextResponse.json({
            success: true,
            message: 'Number announced successfully',
            number
        });
    } catch (error) {
        console.error('Error in number announcement:', error);
        return NextResponse.json(
            { error: 'Internal server error' },
            { status: 500 }
        );
    }
}
```
### Pusherの用意
ストレージ周りは見たので、次にPusherによる通知を行います。
まずはPusherの設定ですが、以下のURL通りに行えば完了するかと思いますので、ここでは共有のみといたします。
https://zenn.dev/hayato94087/articles/0c266563626a27
### Pusherの設定
Pusherの設定が完了したら、シークレットを環境変数ファイルに設定します。
設定が終わったら、`pusher`と`pusher-js`モジュールをインストールします。
インストール後、任意のファイルに以下の記載をします。
```tsx
import PusherServer from 'pusher';
import PusherClient from 'pusher-js';
export const pusherServer = new PusherServer({
    appId: process.env.PUSHER_APP_ID!,
    key: process.env.NEXT_PUBLIC_PUSHER_KEY!,
    secret: process.env.PUSHER_SECRET!,
    cluster: process.env.NEXT_PUBLIC_PUSHER_CLUSTER!,
    useTLS: true,
});
export const pusherClient = new PusherClient(
    process.env.NEXT_PUBLIC_PUSHER_KEY!,
    {
        cluster: process.env.NEXT_PUBLIC_PUSHER_CLUSTER!,
    }
);
```
Pusherに値を送るためのpushSeverと、Pusherとコネクションを張るためのpusherClientを作成しています。
それぞれの使い方を見ていきます。
Pusherに値を送るpushServerはRoute Handllerで以下のように呼び出しています。
```tsx
import { pusherServer } from '@/lib/pusher';
/** 省略 */
export async function POST(request: NextRequest) {
        const { number, gameId } = await request.json()
        await pusherServer.trigger(
            `bingo-game-${gameId}`,
            'new-number',
            {
                number,
                timestamp: new Date().toISOString()
            }
        );
        /** 省略 */
}
```
Pusherに値を送る時は、triggerメソッドを使用します。
第一引数に値のやり取りを行うチャンネル名を指定します。
第二引数にはチャンネル内で使用するイベント名を指定します。
そして、第三引数に実際にやり取りを行うデータを送ります。
これによって、Pusherへ通知を行うための情報を渡せます。
後は、Pusherがtiggerで指定したコネクションを張っているクライアントに対して通知を行います。
具体的には以下のように行います。
```tsx
'use client'
import { useEffect } from 'react';
import { pusherClient } from '@/lib/pusher';
export function BingoBoard({ gameId }: { gameId: string }) {
    const [numbers, setNumbers] = useState<number[]>([]);
    const [currentNumber, setCurrentNumber] = useState<number | null>(null);
    useEffect(() => 
		// コネクションを張るチャンネル名を指定
		const gameChannel = pusherClient.subscribe(`bingo-game-${gameId}`);
        // チャンネル内のイベントを受け取りコールバックを指定
        gameChannel.bind('new-number', (data: { number: number }) => {
            setCurrentNumber(data.number)
            setTimeout(() => {
                setNumbers(prev => [...prev, data.number]);
            }, 3500);
        });
        // コネクションの解除
        return () => {
            pusherClient.unsubscribe(`bingo-game-${gameId}`);
        };
    }, [gameId]);
    /** 省略 */
}
```
予め用意したpusherClientが持つ、subscribeでコネクションを張りたいチャンネルを指定します。
そして、そのチャンネル内でやり取りされるイベントを待ち構え、イベントを検知した時実行するコールバックを登録しておきます。
これによって、Pusherからの通知をフロントエンド側で受け取れるようになります。
実際にやってみた感想ですが、とても簡単でした。
プロジェクト作って、Pusherとのやり取りができるモジュールをインストールしてしまえばほぼ終わりで、ストレスなく実装ができました。
また、一日20万通のメッセージのやり取りができ、同時に100台の接続が無料でできるので、今回のような小規模アプリであれば全然大丈夫でした。
以上ビンゴアプリの主要な機能についてでした。
もしコード全体が見たい方は以下にリンクをおいていますので、確認いただけますと幸いです。(なお、不要なコードとかもかなりあって結構汚いです。ご容赦ください。)
https://github.com/maronnjapan/bingo-game
### 余談 アニメーションについて
ビンゴの番号を出すアニメーションはどうやっているのかというお話もいただいたのですが、ここに関してはひたすら生成AIに頑張ってもらいました。
なので、実装の解説はせずコンポーネントのコードだけおいておきます。
```tsx
import React, { useState, useEffect, useCallback } from 'react';
export const SpineMachine = ({ finalNumber = 42, duration = 3000 }) => {
    const [isSpinning, setIsSpinning] = useState(false);
    const [currentNumber, setCurrentNumber] = useState('00');
    const [drumRotation, setDrumRotation] = useState(0);
    const [ballPosition, setBallPosition] = useState({ x: 100, y: 120 });
    const [showBall, setShowBall] = useState(false);
    const spin = useCallback(() => {
        setIsSpinning(true);
        setShowBall(false);
        setBallPosition({ x: 100, y: 120 });
    }, []);
    useEffect(() => {
        spin()
    }, [finalNumber])
    useEffect(() => {
        if (!isSpinning) return;
        let startTime: number | null = null;
        let animationFrame: number | null = null;
        const animate = (timestamp: number) => {
            if (!startTime) startTime = timestamp;
            const progress = timestamp - startTime;
            const progressRatio = Math.min(progress / duration, 1);
            const easeOut = (t: number) => 1 - Math.pow(1 - t, 3);
            const currentSpeed = easeOut(1 - progressRatio);
            const rotationIncrement = currentSpeed * 30;
            setDrumRotation(prev => (prev + rotationIncrement) % 360);
            if (progress > duration - 1000) {
                setShowBall(true);
                const fallProgress = (progress - (duration - 1000)) / 1000;
                // 自然な落下曲線を計算
                const startX = 155;
                const startY = 160;
                const endX = 100;
                const endY = 270;
                // 重力加速度を考慮した落下
                const x = startX + (endX - startX) * fallProgress;
                const y = startY + (endY - startY) * (fallProgress * fallProgress);
                setBallPosition({ x, y });
                if (progress > duration - 500) {
                    setCurrentNumber(String(finalNumber).padStart(2, '0'));
                }
            } else if (progress % (16 / currentSpeed) < 16) {
                setCurrentNumber(String(Math.floor(Math.random() * 100)).padStart(2, '0'));
            }
            if (progress < duration) {
                animationFrame = requestAnimationFrame(animate);
            } else {
                setIsSpinning(false);
                setCurrentNumber(String(finalNumber).padStart(2, '0'));
            }
        };
        animationFrame = requestAnimationFrame(animate);
        return () => {
            if (animationFrame) {
                cancelAnimationFrame(animationFrame);
            }
        };
    }, [isSpinning, duration, finalNumber]);
    return (
        <div className="flex flex-col items-center justify-center " style={{ marginBottom: 0, marginTop: 0 }}>
            <div className="relative w-44 h-64">
                <svg viewBox="0 0 200 300" className="w-full h-full">
                    {/* ベース */}
                    <rect x="40" y="200" width="120" height="80" fill="#2A4365" rx="5" />
                    <rect x="45" y="205" width="110" height="70" fill="#3B82F6" rx="3" />
                    {/* メインドラム */}
                    <g transform="translate(100 120)">
                        <g transform={`rotate(${drumRotation})`}>
                            <circle cx="0" cy="0" r="60" fill="#1E40AF" />
                            <circle cx="0" cy="0" r="55" fill="#2563EB" />
                            {/* 内部の仕切り */}
                            {[...Array(12)].map((_, i) => (
                                <g key={i} transform={`rotate(${i * 30})`}>
                                    <line
                                        x1="0"
                                        y1="-55"
                                        x2="0"
                                        y2="-35"
                                        stroke="#fff"
                                        strokeWidth="3"
                                        className={isSpinning ? 'opacity-50' : 'opacity-100'}
                                    />
                                </g>
                            ))}
                        </g>
                    </g>
                    {/* 出口の穴 */}
                    <g transform="translate(155, 160)">
                        {/* 穴の外枠 */}
                        <circle cx="0" cy="0" r="8" fill="#1E3A8A" />
                        {/* 穴の内側（奥行き表現） */}
                        <circle cx="0" cy="0" r="6" fill="#152C61" />
                    </g>
                    {/* 受け皿 */}
                    <path
                        d="M70,260 C70,250 130,250 130,260 L130,270 C130,280 70,280 70,270 Z"
                        fill="#2A4365"
                        stroke="#1E3A8A"
                        strokeWidth="1"
                    />
                    {/* ガラスドーム */}
                    <path
                        d="M40,120 A60,60 0 0,1 160,120 L160,200 L40,200 Z"
                        fill="rgba(255,255,255,0.1)"
                        stroke="#A3BFFA"
                        strokeWidth="2"
                    />
                    {/* 落下中のビンゴボール */}
                    {showBall && (
                        <g transform={`translate(${ballPosition.x} ${ballPosition.y})`}>
                            <circle r="6" fill="white" stroke="#2563EB" strokeWidth="1">
                                <animate
                                    attributeName="r"
                                    values="6;5.5;6"
                                    dur="0.5s"
                                    repeatCount="indefinite"
                                />
                            </circle>
                            <text
                                textAnchor="middle"
                                dy="3"
                                fontSize="8"
                                fill="#2563EB"
                                fontWeight="bold"
                            >
                                {currentNumber}
                            </text>
                        </g>
                    )}
                </svg>
                {/* 出てきた玉の最終表示 */}
                {showBall && ballPosition.y > 260 && (
                    <div className="absolute bottom-6 left-0 w-full flex justify-center">
                        <div className="bg-white rounded-full w-12 h-12 flex items-center justify-center shadow-lg border-2 border-blue-600 animate-bounce">
                            <span className="font-mono font-bold text-xl text-blue-600">
                                {currentNumber}
                            </span>
                        </div>
                    </div>
                )}
            </div>
        </div>
    );
};
```
生成AIでも結構頑張ってもらったので、こういのを自前で書いている記事とか見て、改めて尊敬しました。
## なぜビンゴアプリを作ったのか
実装について見ていきましたが、なぜそもそもビンゴアプリを作ったのかについてもお話します。
世にはBINGO! Online（[https://bingo-online.jp/](https://bingo-online.jp/)）をはじめとした多くのビンゴアプリがあります。
なので、わざわざ車輪の再発名をする必要はありません。
ただ、純粋なビンゴアプリならという話ですが。
実は、このビンゴアプリ最初の9回は出る数字が決まっています。
なので、予め当たりの出るカードが用意されていました。
当たりのカードを選ぶという運は必要ですが、一度それを引いてしまえば速攻で当たってしまいます。
これをやりたいがために、ビンゴアプリを作成しました。
一緒に検討していただいた方たちの間では、マッチポンプビンゴと言っていました。
実際ストレートで当たっている方の周りで、豪運という声も聞こえてきて、狙い通りでした。
また、以下のようなエンジニアにはお馴染み？の500エラーを出すという小ネタも入れたかったのも理由です。
![bingo-error-demo.gif](/images/bingo-game-app/bingo-error-demo.gif)
やりたいことが一通りできたので、個人的には満足です。
## 意外と負荷に強かったVercel
今回十数人が同時にアクセスした中での開催だったので、不可がかなり心配でした。
ですが、全然アプリケーションの動きに問題なく、重いといったことが無かったのは意外でした。
なので、今後もし機会があれば、負荷を考慮した結果そぎ落とした機能を復活させ、よりマッチポンプなビンゴアプリ作っていければと思います。(ビンゴアプリを使う機会があればですが…)
ここまで読んでいただきありがとうございました。