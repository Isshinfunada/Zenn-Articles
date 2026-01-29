---
title: "「作って学ぶブラウザのしくみ」第3章：HTTPクライアントの実装とテスト"
type: "tech"
emoji: "🌐"
topics:
  - "ブラウザ"
  - "HTTP"
  - "Rust"
  - "WasabiOS"
published: false
---
# 今日やったこと

「作って学ぶブラウザのしくみ」の第3章を読み進め、HTTP実装のユニットテストによる動作確認と、WasabiOS上でHTTPクライアントを動作させる実装を行いました。具体的には以下の作業を進めました：

- HTTPレスポンス構造体のユニットテスト作成と実行
- HTTPクライアントアプリケーションの作成
- WasabiOS上でのアプリケーション実行
- ローカルテストサーバーの構築とテストHTMLへのアクセス確認

# 今日学んだこと

## ユニットテストによるHTTPレスポンスの検証

HTTPレスポンスが正しく構築されているかを検証するためのユニットテストを作成しました。成功ケースとして以下の4つのパターンをテストしました：

- **ステータスラインのみのレスポンス**: `HTTP/1.1 200 OK` だけの最小構成
- **1つのヘッダーを持つレスポンス**: Dateヘッダーを含むケース
- **2つのヘッダーを持つレスポンス**: DateとContent-Lengthヘッダーを含むケース
- **ボディを含むレスポンス**: ヘッダーとボディの両方を含む完全なレスポンス

また、失敗ケースとして、改行文字がない不正なレスポンス文字列に対してエラーが返されることも確認しました。`is_err`メソッドを使って、`HttpResponse::new`関数からエラーが返ることを検証できます。

## WasabiOS上でのHTTPクライアント実行

`saba_core`ディレクトリに実装された`HttpResponse`構造体はユニットテストで検証できましたが、`net/wasabi`ディレクトリの`HttpClient`構造体はWasabiOSが提供する`noli`ライブラリに依存しているため、WasabiOS上で動作させる必要がありました。

実装中にいくつかの問題に直面しました：

- **Debugトレイトの欠如**: 構造体をデバッグ出力するために必要なトレイトが実装されていなかった（漏れていた）
- **環境変数の引き継ぎ問題**: QEMUフラグとVNCモードに関連する環境変数がサブプロセスに正しく渡されておらず、Wasabi OSのGUIが立ち上がっていなかった

環境変数の問題については、MacOSでは`DISPLAY`環境変数が通常設定されないため、VNCモードで起動してしまい画面が表示されないという現象が起きていました。

### 実際のテストの様子
![[Pasted image 20260129105319.png]]

#### クライアント側
``` sh
❯ ./run_on_wasabi.sh

~~~~~~~~~~~~~~~~

saba
[INFO]  os/src/cmd.rs:58 :  Executing cmd: ["saba"]
[INFO]  os/src/net/manager.rs:221:  socket created: TcpSocket{ state: SynSent }
[INFO]  os/src/net/manager.rs:169:  dynamic TCP port 49152 is picked
[INFO]  os/src/net/tcp.rs:470:  Trying to open a socket with 10.0.2.2:8000
[INFO]  os/src/net/tcp.rs:285:  net: tcp: send: TCP :49152 -> :8000, seq = 1234, ack = 0, flags = 0b0000000000000010 SYN
[INFO]  os/src/syscall.rs:171:  tx data enqueued. waiting...
[INFO]  os/src/net/tcp.rs:306:  net: tcp: recv: TCP :8000 -> :49152, seq = 4864001, ack = 1235, flags = 0b0000000000010010 SYN ACK
[INFO]  os/src/net/tcp.rs:349:  net: tcp: recv: TCP connection established
[INFO]  os/src/net/tcp.rs:285:  net: tcp: send: TCP :49152 -> :8000, seq = 1235, ack = 4864002, flags = 0b0000000000010000 ACK
[INFO]  os/src/net/tcp.rs:429:  Trying to send data [71, 69, 84, 32, 47, 116, 101, 115, 116, 46, 104, 116, 109, 108, 32, 72, 84, 84, 80, 47, 49, 46, 49, 10, 72, 111, 115, 116, 58, 32, 104, 111, 115, 116, 46, 116, 101, 115, 116, 10, 65, 99, 99, 101, 112, 116, 58, 32, 116, 101, 120, 116, 47, 104, 116, 109, 108, 10, 67, 111, 110, 110, 101, 99, 116, 105, 111, 110, 58, 32, 99, 108, 111, 115, 101, 10, 10] to 10.0.2.2:8000
[INFO]  os/src/net/tcp.rs:285:  net: tcp: send: TCP :49152 -> :8000, seq = 1235, ack = 4864002, flags = 0b0000000000010000 ACK
[INFO]  os/src/syscall.rs:177:  write done
[INFO]  os/src/net/tcp.rs:306:  net: tcp: recv: TCP :8000 -> :49152, seq = 4864002, ack = 1312, flags = 0b0000000000010000 ACK
[INFO]  os/src/net/tcp.rs:285:  net: tcp: send: TCP :49152 -> :8000, seq = 1312, ack = 4864002, flags = 0b0000000000010000 ACK
[INFO]  os/src/net/tcp.rs:306:  net: tcp: recv: TCP :8000 -> :49152, seq = 4864002, ack = 1312, flags = 0b0000000000010000 ACK
[INFO]  os/src/net/tcp.rs:285:  net: tcp: send: TCP :49152 -> :8000, seq = 1312, ack = 4864187, flags = 0b0000000000010000 ACK
[INFO]  os/src/net/tcp.rs:306:  net: tcp: recv: TCP :8000 -> :49152, seq = 4864187, ack = 1312, flags = 0b0000000000010000 ACK
[INFO]  os/src/net/tcp.rs:285:  net: tcp: send: TCP :49152 -> :8000, seq = 1312, ack = 4864262, flags = 0b0000000000010000 ACK
[INFO]  os/src/net/tcp.rs:306:  net: tcp: recv: TCP :8000 -> :49152, seq = 4864262, ack = 1312, flags = 0b0000000000010001 FIN ACK
[INFO]  os/src/net/tcp.rs:285:  net: tcp: send: TCP :49152 -> :8000, seq = 1312, ack = 4864263, flags = 0b0000000000010001 FIN ACK
response:
HttpResponse {
    version: "HTTP/1.0",
    status_code: 200,
    reason: "OK",
    headers: [
        Header {
            name: "Server",
            value: "SimpleHTTP/0.6 Python/3.14.2",
        },
        Header {
            name: "Date",
            value: "Thu, 29 Jan 2026 01:51:43 GMT",
        },
        Header {
            name: "Content-type",
            value: "text/html",
        },
        Header {
            name: "Content-Length",
            value: "75",
        },
        Header {
            name: "Last-Modified",
            value: "Tue, 27 Jan 2026 00:33:01 GMT",
        },
    ],
    body: "<html>\n<body>\n    <h1>Test Page</h1>\n    <p>hello world</p>\n</body>\n</html>",
}[INFO]  os/src/cmd.rs:154:  Ok(0)
> [INFO]  os/src/net/tcp.rs:306:  net: tcp: recv: TCP :8000 -> :49152, seq = 4864263, ack = 1313, flags = 0b0000000000010000 ACK
[INFO]  os/src/net/tcp.rs:379:  net: tcp: recv: TCP connection closed
```
## 1. 初期化とソケット作成

```
saba
[INFO]  os/src/cmd.rs:58 :  Executing cmd: ["saba"]
[INFO]  os/src/net/manager.rs:221:  socket created: TcpSocket{ state: SynSent }
[INFO]  os/src/net/manager.rs:169:  dynamic TCP port 49152 is picked
[INFO]  os/src/net/tcp.rs:470:  Trying to open a socket with 10.0.2.2:8000
```

sabaアプリケーションが起動し、TCPソケットを作成。クライアント側のポートとして49152が動的に割り当てられた（エフェメラルポート）。接続先は `10.0.2.2:8000`、つまりQEMUホスト側で動いているPythonサーバー。

## 2. 3ウェイハンドシェイク（コネクション確立）

```
[INFO]  net: tcp: send: TCP :49152 -> :8000, seq = 1234, ack = 0, flags = SYN
[INFO]  net: tcp: recv: TCP :8000 -> :49152, seq = 4864001, ack = 1235, flags = SYN ACK
[INFO]  net: tcp: recv: TCP connection established
[INFO]  net: tcp: send: TCP :49152 -> :8000, seq = 1235, ack = 4864002, flags = ACK
```

TCPの基本、3ウェイハンドシェイク。

1. **SYN**: クライアント→サーバー「接続したい」（seq=1234を提示）
2. **SYN+ACK**: サーバー→クライアント「OK、こちらのseqは4864001。君の1234は受け取った（ack=1235）」
3. **ACK**: クライアント→サーバー「了解」（ack=4864002 = 相手のseq+1）

これで双方向の通信路が確立された。

## 3. HTTPリクエスト送信

```
[INFO]  Trying to send data [71, 69, 84, 32, 47, 116, 101, 115, 116, ...] to 10.0.2.2:8000
[INFO]  net: tcp: send: TCP :49152 -> :8000, seq = 1235, ack = 4864002, flags = ACK
[INFO]  write done
```

バイト列 `[71, 69, 84, 32, ...]` はASCIIで「GET /test.html HTTP/1.1...」。HTTPリクエストがTCPペイロードとして送信された。seq=1235から始まり、77バイト送信したので次のseqは1312になる。

## 4. レスポンス受信（データ転送）

```
[INFO]  net: tcp: recv: TCP :8000 -> :49152, seq = 4864002, ack = 1312, flags = ACK
[INFO]  net: tcp: send: TCP :49152 -> :8000, seq = 1312, ack = 4864002, flags = ACK
[INFO]  net: tcp: recv: TCP :8000 -> :49152, seq = 4864002, ack = 1312, flags = ACK
[INFO]  net: tcp: send: TCP :49152 -> :8000, seq = 1312, ack = 4864187, flags = ACK
[INFO]  net: tcp: recv: TCP :8000 -> :49152, seq = 4864187, ack = 1312, flags = ACK
[INFO]  net: tcp: send: TCP :49152 -> :8000, seq = 1312, ack = 4864262, flags = ACK
```

サーバーからHTTPレスポンスが届いている。ack値の変化を見ると：

- 4864002 → 4864187: 185バイト受信（HTTPヘッダー部分）
- 4864187 → 4864262: 75バイト受信（HTMLボディ、Content-Lengthと一致）

クライアントは受信するたびにACKを返して「ここまで受け取った」と伝えている。
## 5. コネクション終了（4ウェイハンドシェイク簡略版）

```
[INFO]  net: tcp: recv: TCP :8000 -> :49152, seq = 4864262, ack = 1312, flags = FIN ACK
[INFO]  net: tcp: send: TCP :49152 -> :8000, seq = 1312, ack = 4864263, flags = FIN ACK
```

サーバーが `Connection: close` に従ってFINフラグを送信し「もう送るデータない」と通知。クライアントもFIN+ACKで応答し、双方が終了に合意。

```
[INFO]  net: tcp: recv: TCP :8000 -> :49152, seq = 4864263, ack = 1313, flags = ACK
[INFO]  net: tcp: recv: TCP connection closed
```

最後のACKを受け取って接続が正式にクローズ。


#### サーバー側
```sh
❯ python3 -m http.server 8000
Serving HTTP on :: port 8000 (http://[::]:8000/) ...
::ffff:127.0.0.1 - - [29/Jan/2026 10:51:43] "GET /test.html HTTP/1.1" 200 -
```



## localhostとhost.testの違い

ローカルサーバーを構築してテストする際、重要な概念を学びました：

- **localhost**: ネットワーク上の自分自身を指す特殊なホスト。IPv4では`127.0.0.1`、IPv6では`::1`がループバックアドレスとして使用される
- **host.test**: QEMU内のOSからQEMU外のOSのlocalhostにアクセスするために、Wasabi OSで名前解決できるよう実装されている特殊なホスト名

私たちのアプリケーションはQEMU内のOS（WasabiOS）で動作しているのに対し、Pythonで起動したテストサーバーはQEMU外のOS（Mac）で動作しています。そのため厳密には同じローカル環境ではなく、localhostに直接アクセスすることはできません。WasabiOSのサポートにより、`host.test`へのアクセスがQEMU外のOSのlocalhostにフォワーディングされる仕組みが用意されています。

## Rustのトレイトとderiveマクロ

Debugトレイトの問題を調査する中で、Rustのトレイトとderiveの仕組みについて理解を深めました：

- **トレイト**: インターフェースの定義。特定のメソッドを持つべきという仕様を定めるもの
- **derive**: トレイトの実装を自動生成するマクロ。`#[derive(Debug)]`と書くと、構造体にDebugトレイトの実装が自動生成される

deriveマクロは**直下の構造体にのみ適用される**という重要なルールがあります。これはRustの属性が常に対象の直前につくという規則に従っており、コンパイル時に構造体全体をパースしてコードを生成しやすくするための設計です。

# 次にやること

第3章が完了し、HTTPリクエストを送信してローカルサーバーからレスポンスを受け取る機能が実現できました。次回は第4章に進み、HTMLパーサーの実装を学習します。
