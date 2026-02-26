---
title: "Live Serverの代替にFive ServerとwebpackのHTTPS配信を試してみた"
type: "tech"
emoji: "🔒"
topics:
  - "webpack"
  - "HTTPS"
  - "VSCode"
  - "mkcert"
  - "セキュリティ"
published: true
---
## はじめに

VS Codeの**Live Server**拡張、ローカルで静的ファイルを配信するのに便利なので使っている人は多いと思います。自分もkintoneカスタマイズの開発で、ビルドしたJSをローカルから配信するのに使っていました。

ただ、Live Serverには脆弱性が見つかっていて、しかも未パッチのまま放置されています。代替として**Five Server**や**webpack-dev-server**でHTTPS対応のローカルサーバーを立てる方法を試したので、それぞれの特徴と自分が最終的に選んだ構成を紹介します。

## Live Serverの脆弱性（CVE-2025-65717）

2025年8月、Live Server拡張（v5.7.9以降）にファイル窃取の脆弱性（[CVE-2025-65717](https://nvd.nist.gov/vuln/detail/CVE-2025-65717)）が報告されました。

### 何が問題なのか

Live Serverはデフォルトで**CORS保護が実装されていません**。Live Serverが起動している状態で悪意のあるWebページを開くと、そのページから`localhost:5500`へのクロスオリジンリクエストが通ってしまいます。

これを悪用すると：
1. Live Serverが配信しているローカルファイルを再帰的にクロール
2. ソースコード、設定ファイル、認証情報などを外部に送信

という形でファイルを窃取できます。

### パッチ状況

2026年2月時点で、この脆弱性は**未パッチのまま**です。7200万以上のインストール数を持つ拡張機能がこの状態なので、使い続けるのはリスクが高いです。

## 代替案1：Five Server

Live Serverの後継的な立ち位置にある**Five Server**はHTTPSに対応しています。`fiveserver.config.js`を作成して証明書の内容をインラインで貼り付ければ、HTTPS配信が可能です。[パス指定は効かない](https://github.com/yandeu/five-server/discussions/151)ので、中身をまるごと書く必要があります。

```js
// fiveserver.config.js
module.exports = {
  https: {
    cert: `-----BEGIN CERTIFICATE-----
...証明書の中身をまるごと貼る...
-----END CERTIFICATE-----`,
    key: `-----BEGIN PRIVATE KEY-----
...秘密鍵の中身をまるごと貼る...
-----END PRIVATE KEY-----`
  }
}
```

実際に試したところ、これでHTTPS配信はちゃんと動きます。個人でサクッとやるならこれで十分です。

ただ、いくつか気になる点がありました：

- VS Codeの設定（settings.json）からはHTTPSを有効にできず、プロジェクトごとに`fiveserver.config.js`を作る必要がある
- 証明書の中身をインラインで貼るので、GitHubでチーム管理しているリポジトリにはコミットしづらい
- プロジェクトが増えるたびに同じ証明書を貼り回すのが面倒

個人開発でwebpackを使っていないプロジェクトなら、Five Serverは手軽な選択肢だと思います。

## 代替案2：webpack-dev-server + mkcert

[webpack-dev-server](https://webpack.js.org/configuration/dev-server/)にはHTTPS対応が組み込まれていて、証明書のパス指定も普通にできます。[mkcert](https://github.com/FiloSottile/mkcert)でローカル証明書を発行すれば、ブラウザに信頼されたHTTPSサーバーがすぐ立ちます。

### 1. mkcertのインストールと証明書の発行

```bash
# mkcertのインストール（Mac）
brew install mkcert

# ローカルCAをシステムに登録
mkcert -install

# プロジェクトルートで証明書を発行
mkcert localhost 127.0.0.1 ::1
```

カレントディレクトリに`localhost+2.pem`（証明書）と`localhost+2-key.pem`（秘密鍵）が生成されます。証明書ファイルには秘密鍵が含まれるので、`.gitignore`に追加しておきましょう。

```gitignore
*.pem
```

### 2. webpack.config.jsの設定

```js
const path = require('path');
const fs = require('fs');

module.exports = {
  // ... 他の設定 ...
  devServer: {
    static: path.resolve(__dirname, 'dist'),
    server: {
      type: 'https',
      options: {
        key: fs.readFileSync(path.resolve(__dirname, 'localhost+2-key.pem')),
        cert: fs.readFileSync(path.resolve(__dirname, 'localhost+2.pem')),
      },
    },
    port: 8080,
    hot: false,
    headers: {
      'Access-Control-Allow-Origin': 'https://xxxxx.cybozu.com',
    },
  },
};
```

- `server.type: 'https'` でHTTPS有効化
- mkcertで生成した証明書をパスで指定できる
- `Access-Control-Allow-Origin` は外部ドメインからJSを読み込む場合に必要。**`'*'`にすると任意のサイトからlocalhostのファイルにアクセスできてしまい、Live Serverと同じ脆弱性を抱えることになります**。読み込み元のドメインに絞って指定しましょう

### 3. package.jsonのscripts

```json
{
  "scripts": {
    "dev": "webpack serve --mode development"
  }
}
```

`npm run dev` で起動すれば `https://localhost:8080` でファイルが配信されます。ソースを変更すればwebpackが自動で再ビルドするので、ブラウザをリロードするだけで反映されます。

### チーム開発での注意点

mkcertは各開発者のマシンで証明書を発行する仕組みなので、各自が好きな場所で`mkcert`を実行すると証明書の生成先がバラバラになり、webpack.config.jsのパスと合わなくなります。

上の例ではwebpack.config.jsが`__dirname`（プロジェクトルート）の証明書を参照する設計にしているので、Makefileでプロジェクトルートに証明書を生成するようにしておけば、誰が実行しても同じパスに揃います。

```makefile
# Makefile
.PHONY: setup-certs dev

setup-certs:
	@command -v mkcert >/dev/null 2>&1 || (echo "mkcert をインストールしてください: brew install mkcert" && exit 1)
	@test -f localhost+2.pem || (mkcert -install && mkcert localhost 127.0.0.1 ::1)

dev: setup-certs
	npm run dev
```

`make dev` を叩けば、証明書がなければプロジェクトルートに自動生成してからdev serverを起動します。これならREADMEに「`make dev`で起動」と書いておくだけで済みます。私はまだここまでやっていませんが。

## 用途の例：kintoneカスタマイズ開発

自分がこの構成にした直接のきっかけは、kintoneカスタマイズの開発環境でした。

kintoneではJavaScript/CSSカスタマイズでURL指定によるJS読み込みができます。cybozu developer networkでも[Live Serverを使ったローカル開発の方法](https://cybozu.dev/ja/kintone/tips/development/customize/development-know-how/use-visual-studio-code-live-server-extension/)が紹介されています。

ただし、kintoneのURL指定はHTTPSのURLしか受け付けません。HTTPのローカルサーバーだとフォームのバリデーションで弾かれます。webpack-dev-server + mkcertならHTTPSで配信できるので、kintone側で `https://localhost:8080/bundle.js` をURL指定すればそのまま読み込めます。

kintoneに限らず、Service Workerのテストやセキュアコンテキストが必要なWeb APIの開発など、ローカルでHTTPSが欲しい場面全般で使えます。

## まとめ

Live Serverに脆弱性が見つかったので、代替手段を探しました。

- **Five Server**: 個人でサクッとHTTPS配信したいなら手軽。ただし証明書をインラインで貼る必要があるので、チーム開発やプロジェクトが多い場合は管理が面倒
- **webpack-dev-server + mkcert**: 証明書をパス指定できるので複数プロジェクトで使い回しやすい。webpackの自動ビルドもそのまま活かせる

自分の場合はwebpackをすでに使っていて、kintoneカスタマイズの開発で今後複数プロジェクトを扱う想定なので、webpack-dev-server + mkcertの構成がしっくりきました。今のところ快適に使えています。
