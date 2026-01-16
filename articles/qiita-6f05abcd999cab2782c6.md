---
title: "NotionのDBをObsidianに1レコード1ファイルで移行する #Notion"
type: tech
emoji: 📝
topics:
  - Qiita
published: false
---
# NotionのDBをObsidianに1レコード1ファイルで移行する #Notion

> この記事は既存のQiita記事をObsidian管理下に取り込んだものです。

## 元記事情報

- Qiita Item ID: 6f05abcd999cab2782c6
- URL: （status.yamlを参照）

## 本文

### はじめに

NotionのWeb Clipperで記事を収集して、DBとして管理していたのですが、Obsidianで一元管理したくなったため、移行を試みました。目的は、NotionのDBに登録されていた**1レコード1記事を、Obsidianで個別ファイルとして管理できるようにすること**です。

### 最初に試した方法：Importerプラグイン

Obsidianの「Importer」プラグインを試してみました。NotionからHTML形式でエクスポートしたデータを取り込めるのですが、**DBのビューがそのままHTMLとして再現されるだけ**で、各レコードが独立したMarkdownファイルにはなりませんでした。

そのため、ObsidianのZettelkasten的な使い方をしたい自分には合いませんでした。

### やりたかったこと：1レコード＝1ファイル

やりたかったのは、Notionで1レコードとして格納されていた各記事を、**MarkdownファイルとしてObsidianのVault内に個別保存すること**です。各記事を参照したりタグをつけたかったので、1ファイルずつ存在していてほしかったのです。

### 実際の移行方法

結論としては、Notionから「Markdown & CSV」でエクスポートして、`mv`コマンドでObsidianのディレクトリに配置する、という原始的ですがシンプルな方法でうまくいきました。

以下は手順です：

1. Notionで対象のDBを開く
    
2. 「…」メニューから「Export」→「Markdown & CSV」でエクスポート
    
3. 解凍すると、各記事がMarkdownファイルとして保存されている（`PageName.md`など）
    

ObsidianのVaultパスがわからないときは、以下で探せます：

```sh
find ~ -type d -name ".obsidian" 2>/dev/null

```

Vaultの中に `notion_imports` などのフォルダを作って移動します：

```sh
mv ~/Downloads/NotionExport/*.md ~/Library/Mobile\ Documents/.../Obsidian/YourVault/notion_imports/
```

これで、**Obsidian上で各レコードが個別ファイルとして扱えるようになります**。

### おわりに

Importerプラグインでは思ったように移行できませんでしたが、Notionのエクスポート機能だけで、シンプルかつ柔軟に移行できました。

