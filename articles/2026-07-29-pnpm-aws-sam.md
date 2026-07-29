---
title: "ZIP手動デプロイを卒業する pnpm × AWS SAMのハマりポイント"
emoji: "📦"
type: "tech"
topics:
  - "aws"
  - "lambda"
  - "sam"
  - "pnpm"
published: true
---

:::message
この記事は、[CYBOZU SUMMER BLOG FES '26](https://summer-blog-fes.cybozu.io/2026/)の記事です。
:::

## 自己紹介

船長（せんちょう）と申します。

2024年卒のWebエンジニアで、2026年にサイボウズへ中途入社しました。

現在は、サイボウズのパートナー企業向けポータルサイト「CyPN Portal」に適用されている、kintoneカスタマイズの開発・運用・保守を軸に、その周辺領域にも幅広く携わっています。

## この記事の要点

* ZIPファイルを手動アップロードしていたLambdaと周辺リソースをコードで管理し、コマンドでデプロイできる構成へ移行した
* SAMの標準ビルドはnpm前提だったため、Makefile Buildへ切り替え、pnpmでデプロイ用の成果物を作る構成にした
* 責務を決めずにAIへ実装を任せると設定が重複したため、各ファイルの役割と「書かないこと」を整理してやり直した

## 執筆時に確認した環境

執筆時に確認した環境は次のとおりです。

* Lambda Runtime：`nodejs24.x`
* Node.js：`v24.16.0`
* pnpm：`11.5.2`
* AWS SAM CLI：`1.161.0`

## Lambdaと周辺リソースをコードで管理することにした背景

これまで、とあるNode.jsのLambdaでは、アプリケーションコードと依存関係をZIPファイルにまとめ、AWSコンソールから手動でアップロードしていました。

今回、Node.jsランタイムの更新に合わせて、AWS SAMを使ってこのLambdaと周辺リソースをコードで管理し、コマンドでデプロイできる構成へ移行しました。きっかけの一つは、stg環境とprod環境の設定を取り違えて適用したことです。個人の注意だけに頼るのではなく、AWS上のリソースとデプロイ手順をコードで管理し、変更内容を事前に確認できる状態にしたいと考えました。

移行では、主に次のファイルを追加しました。

* `template.yaml`：LambdaやEventBridge、CloudWatch AlarmなどのAWSリソースを定義する
* `samconfig.toml`：環境別のParameter値やSAM CLIの設定を管理する
* Makefile：ビルド、差分確認、デプロイの操作をまとめる

最終的には、次のようなコマンドで一連の操作を行えるようにしています。

```console
make validate
make diff-stg
make deploy-stg
```

この記事では、移行中に詰まった「pnpmプロジェクトのビルド」と「AIへ実装を任せるときの責務分担」に加え、移行後の運用で工夫した点を紹介します。

## AWS SAMへの移行で詰まったところ

### npm前提の標準ビルドをMakefile Buildで回避した

AWS SAMのNode.js向け標準ビルドでは、`package.json`の指定からパッケージマネージャーが自動選択されるわけではなく、npmが実行されます。

このプロジェクトでは、`devEngines.packageManager`にpnpmを指定し、pnpm以外での実行をエラーにしていました。最初はSAM側もこの指定に合わせてpnpmを使ってくれると思っていましたが、実際にはnpmが実行され、ビルドに失敗しました。

このときは、次のようなエラーが発生しました。

```
Error: NodejsNpmBuilder:NpmInstall - NPM Failed
npm error code EBADDEVENGINES
npm error Invalid name "pnpm" does not match "npm" for "packageManager"
```

仮にこの制約を外してnpmを実行できるようにしても、npmは`pnpm-lock.yaml`を使って依存関係を解決しません。加えて、部内ではNode.jsプロジェクトのパッケージマネージャーに原則としてpnpmを使うルールがあり、npmへ切り替える選択肢はありませんでした。そのため、SAMに合わせてnpmを使うのではなく、SAM側のビルド方法をプロジェクトに合わせる必要がありました。

そこで、SAMのMakefile Buildを利用しました。`template.yaml`では、対象のLambdaに`BuildMethod: makefile`を指定します。

```yaml
Resources:
  Function:
    Type: AWS::Serverless::Function
    Metadata:
      BuildMethod: makefile
    Properties:
      CodeUri: .
      Handler: index.handler
      Runtime: nodejs24.x
```

Makefileには、リソースのLogical IDに対応する`build-Function`ターゲットを用意します。SAMから渡される`ARTIFACTS_DIR`へソースコードと本番依存関係を配置し、Lambdaへアップロードする成果物を作ります。

以下は、依存関係を配置する処理だけを抜き出した例です。実際のMakefileでは、ソースコードの配置処理とプロジェクト固有のpnpmオプションも指定しています。

```makefile
build-Function:
	@mkdir -p "$(ARTIFACTS_DIR)"
	@CI=true pnpm install \
		--prod \
		--frozen-lockfile \
		--dir "$(ARTIFACTS_DIR)"
```

これにより、SAMが期待する場所へ成果物を配置しつつ、依存関係の解決にはプロジェクトで利用しているpnpmと`pnpm-lock.yaml`を使えるようになりました。

### 責務を決めずにAIへ任せると設定が複数のファイルに重複した

今回の移行では、AI（Codex 5.5）にも実装を手伝ってもらいました。しかし、最初に「このLambdaをSAM管理へ移行したい」とだけ依頼したときは、期待した構成にはなりませんでした。

たとえば、環境別の設定値が`template.yaml`と`samconfig.toml`の両方へ書かれたり、Makefileにも設定値や不要なコマンドが追加されたりしました。一見すると動きそうでも、どのファイルを変更すればよいのか分かりにくく、運用しづらい構成でした。

振り返ると、「SAM管理へ移行したい」という指示では、手段しか伝えられていませんでした。どのようなビルドやデプロイ運用を目指すのか、自分の中でも完成形を整理できていなかったのです。そこで、やり直す際は各ファイルの責務を次のように整理しました。

| ファイル | 書くもの |
| :-- | :-- |
| `template.yaml` | AWSリソースの骨組みとParameterの定義 |
| `samconfig.toml` | 環境別のParameter値とSAM CLIの設定 |
| `Makefile` | ビルド、差分確認、デプロイの操作 |

特に、環境別のParameter値は`samconfig.toml`だけに置き、AWSリソースの骨組みは`template.yaml`だけに書くことをルールにしました。この境界を崩さないことが、設定の置き場所に迷わないために重要でした。

以下のコードは、実際の構成を説明用に単純化した例です。

#### template.yamlにはAWSリソースの骨組みとParameterの定義を書く

`template.yaml`には、環境間で共通するリソース構成と、環境ごとに受け取るParameterを定義します。`samconfig.toml`の`parameter_overrides`で渡したParameterの値を、`!Ref`で参照します。

```yaml
Parameters:
  NodeEnv:
    Type: String
    AllowedValues:
      - development
      - production
  MemorySize:
    Type: Number
  Timeout:
    Type: Number
  ScheduleExpression:
    Type: String
  ScheduleState:
    Type: String
    AllowedValues:
      - ENABLED
      - DISABLED
  LogRetentionInDays:
    Type: Number

Resources:
  Function:
    Type: AWS::Serverless::Function
    Metadata:
      BuildMethod: makefile
    Properties:
      FunctionName: !Ref AWS::StackName
      CodeUri: .
      Handler: index.handler
      Runtime: nodejs24.x
      MemorySize: !Ref MemorySize
      Timeout: !Ref Timeout
      Environment:
        Variables:
          NODE_ENV: !Ref NodeEnv

  InvocationSchedule:
    Type: AWS::Events::Rule
    Properties:
      ScheduleExpression: !Ref ScheduleExpression
      State: !Ref ScheduleState
      Targets:
        - Arn: !GetAtt Function.Arn
          Id: Function

  FunctionLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub "/aws/lambda/${AWS::StackName}"
      RetentionInDays: !Ref LogRetentionInDays
```

#### samconfig.tomlには環境別のParameter値とSAM CLIの設定を書く

`samconfig.toml`には、スタック名やParameterの具体的な値など、SAM CLIが環境ごとに使用する設定を書きます。

```toml
[stg.deploy.parameters]
stack_name = "example-lambda-stg"
parameter_overrides = [
  "NodeEnv=development",
  "MemorySize=128",
  "Timeout=3",
  'ScheduleExpression="rate(5 minutes)"',
  "ScheduleState=DISABLED",
  "LogRetentionInDays=7",
]

[prod.deploy.parameters]
stack_name = "example-lambda-prod"
parameter_overrides = [
  "NodeEnv=production",
  "MemorySize=1024",
  "Timeout=120",
  'ScheduleExpression="rate(5 minutes)"',
  "ScheduleState=ENABLED",
  "LogRetentionInDays=7",
]
```

#### Makefileにはビルド・差分確認・デプロイの操作を書く

Makefileには、ビルド、差分確認、デプロイのコマンドをまとめます。

```makefile
build:
	sam build

diff-stg: build
	sam deploy --config-env stg --no-execute-changeset

deploy-stg: build
	sam deploy --config-env stg
```

AIへも、この責務分担と「各ファイルの責務の範囲を越える内容は書かない」というルールを明示しました。生成されたコードについても、設定値や処理が複数箇所へ重複していないかを確認しながら修正しました。

AIを使うことで実装は速く進められますが、運用しやすい完成形の判断基準まで自動的に決めてくれるわけではありません。AIへ任せる場合ほど、自分自身で目指す構成と責務分担を整理しておく必要があると学びました。

## 移行後の運用で工夫したこと

### デプロイ前の変更をChange SetとParameter差分で確認する

移行後は、デプロイ前に変更内容を確認できるようにしました。

`make diff-*`では、`sam deploy --no-execute-changeset`でChange Setを作成し、リソース差分を表示します。さらに、現在のCloudFormation Parameterと、これから渡すParameterをunified diff形式で表示します。

動作確認のため、stgのParameterを意図的にいくつか変更してから`make diff-stg`を実行しました。値は記事用のサンプルに置き換えています。

```console
$ make diff-stg

sam build
...
Build Succeeded
...
CloudFormation stack changeset
--------------------------------------------------------------------------------
Operation      LogicalResourceId      ResourceType             Replacement
--------------------------------------------------------------------------------
* Modify       Function               AWS::Lambda::Function    False
* Modify       FunctionLogGroup       AWS::Logs::LogGroup      False
--------------------------------------------------------------------------------

Changeset created successfully.

=== Parameter diff (current -> new) ===
--- current
+++ new
@@ -1,4 +1,4 @@
-EphemeralStorageSize=512
-LogRetentionInDays=7
-MemorySize=128
-Timeout=3
+EphemeralStorageSize=1024
+LogRetentionInDays=14
+MemorySize=1024
+Timeout=120
```

Change Setのリソース差分だけでなく、後続のParameter差分を見ることで、どの値を変更しようとしているのかもまとめて確認できます。`make diff-*`ではChange Setを実行しないため、この変更は環境へ適用されていません。

## まとめ：手動デプロイから、再現可能で変更を確認できるデプロイへ

今回の移行により、Lambdaと周辺リソースの構成、ビルド方法、デプロイ手順をリポジトリで管理できるようになりました。

AWS SAMを導入すること自体が目的ではありません。目指したのは、コマンド一つでデプロイでき、その前に変更内容を確認できる状態です。そのために、pnpmと`pnpm-lock.yaml`を使った再現可能なビルドも含めて整えました。

移行を通じて、SAMの標準ビルドとプロジェクトで使うパッケージマネージャーをどうつなぐか、またAIへ実装を任せる前に完成形と各ファイルの責務を整理することの重要性を学びました。

同じように、pnpmで管理しているLambdaを、AWS SAMでコード管理しようとしている方にとって、少しでも参考になれば嬉しいです。

ここまで読んでいただき、ありがとうございました！

## 参考リンク

* [Default build with AWS SAM - AWS Serverless Application Model](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-using-build.html)
* [Building Lambda functions with custom runtimes in AWS SAM - AWS Serverless Application Model](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/building-custom-runtimes.html)
