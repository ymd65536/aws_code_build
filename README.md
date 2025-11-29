# 【AWS】手を動かして学ぶAWS AWS CodeBuild

## はじめに

この記事ではAWS AWS CodeBuild(以下、本文ではCodeBuild)を使って、アプリケーションをビルドする話を書きます。
主な内容としては実践したときのメモを中心に書きます。（忘れやすいことなど）誤りなどがあれば修正していく想定です。

## AWS CodeBuild とは

今回動かすCodeBuildとはどんなものなのでしょうか。公式ドキュメントでは以下のように説明されています。

> AWS CodeBuild は、 クラウドでのフルマネージドビルドサービスです。CodeBuild はソースコードをコンパイルし、単体テストを実行して、すぐにデプロイできるアーティファクトを生成します。

[引用：とは AWS CodeBuild - AWS CodeBuild](https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/welcome.html)

また、公式サイトでは以下のように説明されています。

> 自動スケーリングを使用したコードのビルドとテスト

[引用：マネージド型ビルドサーバー – AWS CodeBuild – AWS](https://aws.amazon.com/jp/codebuild/)

つまりはクラウド上で管理が不要なビルドサーバーを提供するサービスです。
一時的なビルドサーバーを立ち上げ、ビルドアーティファクトを生成できたらサーバーを破棄するような使い方ができます。

※アーティファクト：ビルドやコンパイルの結果生成される成果物のこと。一般的な翻訳だと人工物などの意味がありますが、IT用語としてはビルド成果物のことを指します。

### 余談：AWSのCode兄弟

公的な呼び名ではないですが、AWSにはCodeBuild以外にもCodeという名前がつくサービスがあります。
これらをまとめて俗に「AWSのCode兄弟」と呼ぶことがあります。※私も勝手にそう呼んでいます。

それぞれの役割としては以下の通りです。

- CodeCommit：ソースコードのバージョン管理を行うサービス（Gitリポジトリのホスティングサービス）
- CodeBuild：ソースコードのビルドとテストを行うサービス
- CodeArtifact：ビルド成果物の保存と管理を行うサービス、パッケージレジストリ
- CodeDeploy：アプリケーションのデプロイを自動化するサービス
- CodePipeline：CodeCommit、CodeBuild、CodeDeployなどのサービスを連携させて、継続的インテグレーションと継続的デリバリー（CI/CD）を実現するサービス

このCode兄弟を組み合わせることで、ソースコードの管理からビルド、テスト、デプロイまでの一連のプロセスをAWS内で自動化できます。

## ハンズオン

説明は以上にして、CodeBuildを使って簡単なビルドを実行してみます。今回はGitHubのパブリックリポジトリを使った手順になります。具体的には以下の通りです。

- 事前準備
  - buildspec.ymlを用意する
  - ECRでリポジトリを作成する

- CodeBuildの設定と実行
  - CodeBuild用のIAMロールを作成する
  - CodeBuildプロジェクトを作成する
  - CodeBuildを実行する
  - ビルド結果を確認する

## AWS CodeBuildと合わせて使うサービス

ハンズオンは以上です。ここからはCodeBuildを他のサービスと組み合わせて使う場合のパターンを紹介します。
CodeBuildは前述のCode兄弟を組み合わせることを基本とし、他のサービスと連携することでより効果的に利用できます。

以下はCodeBuildとよく組み合わせて使われるAWSサービスの例です。他にもあるとは思いますが、代表的なものを挙げます。
今回は5パターンを紹介します。

### パターン1：静的Webサイトホスティングの場合

- CodeCommit: ソースコードのリポジトリとして使用し、CodeBuildがこのリポジトリからコードを取得してビルドを実行します。
- CodeBuild: ソースコードのビルドを実行します。例えば、静的Webサイトの場合、HTML、CSS、JavaScriptのビルドや最適化を行います。
- S3: ビルド成果物の保存先として使用します。CodeBuildはビルド後に生成されたアーティファクトをS3バケットにアップロードできます。

このパターンでは、CodeBuildがソースコードをビルドし、生成された静的WebサイトのファイルをS3にアップロードします。
S3バケットは静的Webサイトホスティング用に設定され、ユーザーはS3のURLを通じてWebサイトにアクセスできます。

多くの場合では、CloudFrontを組み合わせて使用してコスト効率と可用性を高めることが多いでしょう。

### パターン2: コンテナ化されたアプリケーションのビルドとデプロイの場合

- CodeCommit: ソースコードとDockerfileのリポジトリとして使用します。
- CodeBuild: Dockerイメージのビルドを実行します。
- Amazon ECR: ビルドされたDockerイメージの保存先として使用します
- Amazon ECSまたはAmazon EKS: ビルドされたDockerイメージをデプロイするために使用します。

このパターンでは、CodeBuildがDockerイメージをビルドし、ECRにプッシュします。その後、ECSまたはEKSがそのイメージを使用してコンテナを起動します。
フロントエンドからへのアクセスはALB(Application Load Balancer)を使用して行うことが一般的です。

### パターン3: コンテナ化されたアプリケーションのビルドとデプロイの場合(App Runner利用時)

- CodeCommit: ソースコードとDockerfileのリポジトリとして使用します。
- CodeBuild: Dockerイメージのビルドを実行します。
- Amazon ECR: ビルドされたDockerイメージの保存先として使用します
- AWS App Runner: ビルドされたDockerイメージをデプロイするため

このパターンではアプリケーションに利用するインフラに対して責任共有モデルの範囲が非常に狭く、App Runnerが多くの責任を引き受けます。
つまり、インフラ管理の負担が大幅に軽減されるので、開発者はコードとコンテナイメージの管理に集中できます。

### パターン4: コンテナ化されたアプリケーションのビルドとデプロイの場合(ECS利用時)

- CodeCommit: ソースコードとDockerfileのリポジトリとして使用します。
- CodeBuild: Dockerイメージのビルドを実行します。
- CodeDeploy: ビルドされたDockerイメージをデプロイするために使用します。
- Amazon ECS: ビルドされたDockerイメージをデプロイするため

このパターンでは、CodeDeployを使用してECSサービスのデプロイを自動化します。ALBを必要とする場合も多いでしょう。
なお、リニアデプロイやカナリアデプロイについては最近のアップデートのこともあってECSサービスで対応可能になっています。

[参考：Amazon ECS でリニアデプロイとカナリアデプロイが標準機能としてサポート開始](https://aws.amazon.com/jp/about-aws/whats-new/2025/10/amazon-ecs-built-in-linear-canary-deployments/)

また、デプロイ先にはEC2を使うことも可能です。

### パターン5: コンテナ化されたアプリケーションのビルドとデプロイの場合(Lambda利用時)

- CodeCommit: ソースコードとDockerfileのリポジトリとして使用します。
- CodeBuild: Dockerイメージのビルドを実行します。
- CodeDeploy: ビルドされたDockerイメージをデプロイするために使用します。
- AWS Lambda: ビルドされたDockerイメージをデプロイするため

このパターンではAWS SAMを含むサーバーレスフレームワークを利用することが多いです。
必要に応じてAPI Gatewayを組み合わせて使用します。

## VPC上でAWS CodeBuildを利用する

## おまけ：GitHub Actionsの関係

## AWS CLI インストールと SSO ログイン手順 (Linux環境)

このガイドでは、Linux環境でAWS CLIをインストールし、AWS SSOを使用してログインするまでの手順を説明します。

## 前提条件

- Linux環境（Ubuntu、CentOS、Amazon Linux等）
- インターネット接続
- 管理者権限（sudoが使用可能）
- AWS SSO が組織で設定済み
- Python 3.12.1

## AWS CLI のインストール

### 公式インストーラーを使用（推奨）

最新版のAWS CLI v2を公式インストーラーでインストールします。

```bash
# 1. インストーラーをダウンロード
curl "https://awscli.amazonaws.com/awscli-exe-linux-$(uname -m).zip" -o "awscliv2.zip"

# 2. unzipがインストールされていない場合はインストール
sudo apt update && sudo apt install unzip -y  # Ubuntu/Debian系
# または
sudo yum install unzip -y                     # CentOS/RHEL系

# 3. ダウンロードしたファイルを展開
unzip awscliv2.zip

# 4. インストール実行
sudo ./aws/install

# 5. インストール確認
aws --version

# ダウンロードしたzipファイルと展開したディレクトリを削除してクリーンアップします。
rm  "awscliv2.zip"

# 解凍したディレクトリを削除
rm -rf aws
```

## AWS SSO の設定とログイン

### 1. AWS SSO の設定

AWS SSOを使用するための初期設定を行います。

```bash
aws configure sso
```

設定時に以下の情報の入力が求められます：

- **SSO start URL**: 組織のSSO開始URL（例：`https://my-company.awsapps.com/start`）
- **SSO Region**: SSOが設定されているリージョン（例：`us-east-1`）
- **アカウント選択**: 利用可能なAWSアカウントから選択
- **ロール選択**: 選択したアカウントで利用可能なロールから選択
- **CLI default client Region**: デフォルトのAWSリージョン（例：`ap-northeast-1`）
- **CLI default output format**: 出力形式（`json`、`text`、`table`のいずれか）
- **CLI profile name**: プロファイル名（`default`とします。）

### 2. AWS SSO ログイン

設定完了後、以下のコマンドでログインを実行します。

```bash
aws sso login
```

ログイン時の流れ：
1. コマンド実行後、ブラウザが自動的に開きます
2. AWS SSOのログインページが表示されます
3. 組織のIDプロバイダー（例：Active Directory、Okta等）でログイン
4. 認証が成功すると、ターミナルに成功メッセージが表示されます

### 3. ログイン状態の確認

認証情報を確認します。

```bash
aws sts get-caller-identity
```

正常にログインできている場合、以下のような情報が表示されます：

```json
{
    "UserId": "AROAXXXXXXXXXXXXXX:username@company.com",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/RoleName/username@company.com"
}
```

## トラブルシューティング

### よくある問題と解決方法

#### 1. ブラウザが開かない場合

```bash
# 手動でブラウザを開く場合のURL確認
aws sso login --no-browser
```

表示されたURLを手動でブラウザで開いてください。

#### 2. セッションが期限切れの場合

```bash
# 再ログイン
aws sso login
```

#### 4. プロキシ環境での設定

プロキシ環境の場合、以下の環境変数を設定してください：

```bash
export HTTP_PROXY=http://proxy.company.com:8080
export HTTPS_PROXY=http://proxy.company.com:8080
export NO_PROXY=localhost,127.0.0.1,.company.com
```

## セキュリティのベストプラクティス

1. **定期的な認証情報の更新**: SSOセッションには有効期限があります。定期的に再ログインを行ってください。

2. **最小権限の原則**: 必要最小限の権限を持つロールを使用してください。

3. **プロファイルの分離**: 本番環境と開発環境で異なるプロファイルを使用してください。

4. **ログアウト**: 作業終了時は適切にログアウトしてください：
   ```bash
   aws sso logout --profile <プロファイル名>
   ```

## 参考リンク

- [AWS CLI ユーザーガイド](https://docs.aws.amazon.com/cli/latest/userguide/)
- [AWS SSO ユーザーガイド](https://docs.aws.amazon.com/singlesignon/latest/userguide/)
- [AWS CLI インストールガイド](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
