---
title: "cluster から OSC を送信して Switchbot を操作する"
emoji: "⚙️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "cluster", "golang", "switchbot"]
published: true
---

# はじめに

こんにちは Smith です、クラスター株式会社のプラットフォーム事業部でプロダクトマネージャーとかをやっています。
クラスター株式会社では、バーチャル空間向けのコンテンツを作ったり誰かが作った様々なコンテンツを楽しめる [cluster](https://cluster.mu/) というサービスを提供しています。

cluster では、ユーザーが自らスクリプトを用いてインタラクティブな体験を空間内に作り出せます。
これにより、ただの「鑑賞型」ではない空間設計が可能になります。
（もちろん、スクリプトを書かないで作る方法もあります）

この記事では cluster から OSC 通信をする方法について解説します。

cluster でなにかをつくるならまずここを見よう
https://creator.cluster.mu/


# OSC について

OSC（Open Sound Control）は、UDPをベースにした通信プロトコルで、構造化されたメッセージを送受信する仕組みです。もともとは音響・マルチメディア制御用として開発されましたが、今ではインタラクティブアート、IoT、ロボット制御など幅広い分野で活用されています。

OSC
https://opensoundcontrol.stanford.edu/


# cluster での OSC

cluster では 2025/6/23 のアップデートから cluster の外部に OSC データを送信できるようになりました。

これは、 Cluster Creator Kit という cluster 上でのものづくりをするためのツールの、スクリプト機能を活用することで実現されます。

OSC 送信機能がリリースされたバージョンのリリースノート
https://note.com/cluster_official/n/n8f66d50d64f9


# このサンプルについて

この記事で紹介するのは、cluster 内のユーザーの操作から現実世界の SwitchBot スマートロックを制御するサンプルです。
すべてのコードは gist にて公開されています。
cluster 外が golang なのは単なる好みです。

https://gist.github.com/dolow/b82e937d7d78ea21c56765ca857c59de

ユーザーがバーチャル空間内のオブジェクトを操作すると、そのイベントが OSC メッセージとしてローカルネットワーク上のサーバーに送られ、最終的には SwitchBot の API を通じて物理的な鍵が「ロック」または「アンロック」されます。

このように、仮想と現実をつなぐインタフェースとして OSC が機能している点が大きなポイントです。

以下は Smith 宅のスマートロックデバイスを cluster から操作しているデモです。
https://x.com/do_low/status/1937056520975593492


# サンプルの全体構成

このサンプルは、以下の4つのコンポーネントで構成されています。

1. **osc_sender.js（ClusterScript）**
    - ユーザーのインタラクトをトリガーに、状態（ロック/アンロック）を切り替えて PlayerScript に送信。

2. **pcx_osc_sender.js（PlayerScript）**
    - `osc_sender.js` から受信したロック/アンロックの状態に基づいて、OSC メッセージを生成して外部に送信。

3. **server.go（OSC サーバー）**
    - メッセージを受信し、受信したデータに基づいて `lock/unlock` コマンドを SwitchBot API に送信。

4. **switchbot.go（Switchbot API クライアント）**
    - SwitchBot API の署名認証付きリクエストを構築し、デバイスの検出やコマンド送信を行う。

![全体構成](/images/cluster-osc-sender/figure1.png)

OSC データがどこに送信されるかは、クリエイターではなくて PlayerScript がアタッチされるユーザー側が設定します。
また、ユーザー側の環境に上記の OSC サーバーや Switchbot API クライアントはなければ動作しません。
勝手に他人の家の鍵を開けられるようなヤバいシロモノではありませんので悪しからず。

# コード解説

### osc_sender.js
バーチャル空間内で、ユーザーがアイテムにインタラクトしたタイミングで状態を切り替え、その状態（`true` or `false`）を PlayerScript に送信します。

@[gist](https://gist.github.com/dolow/b82e937d7d78ea21c56765ca857c59de?file=osc_sender.js)


### pcx_osc_sender.js
osc_sender.js から受信したデータを元に、OSC メッセージとして外部に送信します。

@[gist](https://gist.github.com/dolow/b82e937d7d78ea21c56765ca857c59de?file=pcx_osc_sender.js)


### server.go（一部のみ抜粋）
golang で書かれたOSCサーバ。
`/smartlock` アドレスのメッセージを受信し、SwitchBot APIを呼び出す関数に渡します。

```go
const SmartLockPath = "/smartlock"

func main() {
    // 環境変数を読み込んでいる部分

    d := osc.NewStandardDispatcher()
    d.AddMsgHandler(SmartLockPath, func(msg *osc.Message) {
        for _, arg := range msg.Arguments {
            if b, ok := arg.(bool); ok {
                command(token, secret, b)
            }
        }
    })

    // サーバーを起動している部分
}

func command(token string, secret string, lock bool) {
    // スマートロックデバイスにロック/アンロックのコマンドを実行する関数
}

func cacheSmartLockDevice(ctx *context.Context, token string, secret string) error {
    // 登録済みのスマートロックデバイスを抽出し、キャッシュする関数
}
```


### switchbot.go（一部のみ抜粋）
SwitchBot API と通信するためのクライアントコードで、署名付きHTTPリクエストを生成し、デバイス制御を行います。
API 仕様に準拠するため、この中で HMAC 署名の生成や、UUID・タイムスタンプの処理なども行っています。
ここでは関数の概要のみ軽く触れます。

```go
func SearchDevices(ctx *context.Context, deviceType DeviceType, token string, secret string) ([]Device, error) {
    // 登録済みのデバイス一覧を取得します
}
func Command(ctx *context.Context, deviceId DeviceId, token string, secret string, command CommandBody) ([]byte, error) {
    // 任意のコマンドを任意のデバイスに対して実行します
}
func request(ctx *context.Context, method string, path string, token string, secret string, body string) (*http.Response, error) {
    // Switchbot API リクエストを処理する関数です
}
func hmacSHA256String(message, key string) string {
    // HMAC署名を生成する関数です
    // Switchbot API の認証に用いられます
}
```


---

## おわりに

OSC sender の提供開始により、cluster 上での操作が現実世界の IoT デバイスに連携する「バーチャル×フィジカル」のユースケースがより簡単に実現できるようになりました。

バーチャル空間を超えて現実のユーザー環境に干渉する。この構造を用いた体験やユースケースの発明は、クリエイターやデベロッパーの皆さんの本懐かと思います。

今後とも cluster を良き遊び場としてご利用ください。
