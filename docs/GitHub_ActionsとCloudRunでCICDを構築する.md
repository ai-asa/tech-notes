皆さん、GitHub Actionsは利用していますか？
こんにちは。最近CI/CDの概念を知って勉強し始めたASAです。

GitHub Actionsを利用することで、コードのプッシュから本番環境へのデプロイまでを自動化し、開発サイクルを効率化できます。

本記事では、最近CI/CDを学び始めた初学者の方に向けて、GitHub ActionsとGoogle Cloud Runを組み合わせたCI/CDパイプラインの構築方法について、基礎的な手順をご紹介します。

# GitHub ActionsによるCI/CDパイプラインの流れ
![](https://storage.googleapis.com/zenn-user-upload/482350a0983f-20250319.png)

1. **コードのプッシュ**
   - 開発者がGitHubリポジトリにコードをプッシュ
   - GitHub Actionsのワークフローが自動的に起動

2. **コンテナイメージのビルド**
   - GitHub Actionsがソースコードからコンテナイメージを作成
   - Dockerfileに基づいてアプリケーションをビルド

3. **イメージの保存**
   - ビルドされたコンテナイメージをArtifact Registryにプッシュ
   - バージョン管理されたイメージとして保存

4. **Cloud Runへのデプロイ**
   - Artifact Registryから最新のイメージを取得
   - Cloud Runサービスとしてデプロイ
   - 自動的にHTTPSエンドポイントを生成

このパイプラインでは、以下のGCP APIを使用します：

- **Cloud Run API**
  - コンテナアプリケーションの実行環境
  - HTTPSエンドポイントの提供
  - 自動スケーリングの管理

- **Artifact Registry API**
  - コンテナイメージの保存
  - バージョン管理
  - イメージの配布

:::message
本記事では、基本的なDockerイメージの作成やCloud Runへのデプロイ方法は理解済みの方に向けて、GCPとGitHub ActionsのCI/CDパイプラインの構築に焦点を当てて解説します。
:::

# GCP(Google Cloud Platform)側の準備
- プロジェクトの作成 (割愛)
- 必要なAPIの有効化
  - Cloud Run API
  - Container Registry API
- サービスアカウントの作成と権限付与

## プロジェクトの作成
基本的な部分ですので詳細は割愛しますが、[Google Cloud Console](https://console.cloud.google.com/)にてデプロイ先となるGCPプロジェクトを作成してください。また、**プロジェクトIDはGitHub Actionsの設定で使用するためメモしておいてください。**

## 必要なAPIの有効化
以下のAPIを有効化する必要があります:

1. **Cloud Run API**:
   サーバーレスコンテナを実行するために必要。

2. **Artifact Registry API**:
   コンテナイメージを保存しバージョン管理するために必要。

## Artifact Registryリポジトリの作成
コンテナイメージを保存するためのリポジトリを作成します:

1. **Artifact Registryコンソールにアクセス**:
   - メニューから「Artifact Registry」→「リポジトリ」を選択
   - 「リポジトリを作成」をクリック

2. **リポジトリの設定**:
   - 名前を設定（例: `my-app`）
   - 形式: 「Docker」を選択
   - ロケーションタイプ: 「リージョン」を選択
   - リージョン: サービスを提供する地域を選択（例: `asia-northeast1`）
   - 「作成」をクリック
![](https://storage.googleapis.com/zenn-user-upload/4ab05b65e043-20250319.png)
## サービスアカウントへ追加する権限（ロール）
GitHub ActionsからGCPリソースにアクセスするために、サービスアカウントに以下のロールを付与します：

| ロール名 | ロールID | 説明 |
|:---|:---|:---|
| Cloud Run 管理者 | roles/run.admin | Cloud Runサービスのデプロイと管理<br>サービスの作成、更新、削除権限を含む |
| サービス アカウント ユーザー | roles/iam.serviceAccountUser | サービスアカウントとしての実行権限<br>デプロイプロセス中にサービスアカウントを利用するために必要 |
| Artifact Registry 編集者 | roles/artifactregistry.writer | コンテナイメージのプッシュ権限<br>CI/CDパイプラインでビルドしたイメージを登録するために必要 |

**サービスアカウントのJSONキーの内容は後でGitHub Secretsに登録するので、ファイルをダウンロードしておいてください。**

# GitHub側の準備
- リポジトリの作成
- 認証情報の設定（Secrets）
- Dockerfileの作成
- GitHub Actionsワークフローの作成

## リポジトリの作成
Cloud Runにデプロイするための最小構成は以下の通りです：

```
your-project/
├── .github/
│   └── workflows/
│       └── deploy.yml      # GitHub Actionsの設定ファイル
├── main.py                 # アプリケーションのメインファイル
├── .gitignore             # Git管理対象外のファイル設定
├── Dockerfile             # コンテナイメージの定義
└── requirements.txt       # Pythonパッケージの依存関係
```

各ファイル・ディレクトリの役割：
- `.github/workflows/`: GitHub Actionsの自動化ワークフローを定義するディレクトリ
- `main.py`: アプリケーションのエントリーポイント
- `Dockerfile`: コンテナイメージのビルド手順を定義
- `requirements.txt`: Pythonアプリケーションの依存パッケージとそのバージョンを管理

## シークレットを設定
GitHub ActionsからGCPにアクセスするための認証情報をシークレットに登録します。

1. GCPのサービスアカウントキー（JSONファイル）を取得
2. GitHubリポジトリの「Settings」→「Secrets and variables」→「Actions」を開く
3. 「New repositry secret」をクリックして新しいシークレットを作成する
4. 以下の2つのシークレットを任意の名前で追加：
   - **GCPのプロジェクトID**
   - **サービスアカウントキーのJSON内容**をそのまま貼り付け
![](https://storage.googleapis.com/zenn-user-upload/a86002ae3ab8-20250319.png)

## Dockerfileの作成
Cloud Runにて実行するコンテナ環境を構築するためのDockerfileを作成します。

以下はPythonアプリケーションの例です：

```dockerfile
# ベースイメージの指定
FROM python:3.11-slim

# 作業ディレクトリの設定
WORKDIR /app

# 依存関係のインストール
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# アプリケーションコードのコピー
# 最後に実行することで、コードの変更時のビルドを高速化
COPY . .

# 環境変数の設定
# Cloud Runは$PORTで指定されたポートでリッスンすることを要求
# 8080はCloud Runのデフォルトポート
ENV PORT=8080

# コンテナ起動時の実行コマンド
CMD ["python", "main.py"]
```

## GitHub Actionsワークフロー定義ファイルの作成
GitHub Actionsのデプロイメントプロセスを定義し、CI/CDパイプラインの自動化を行うための定義ファイルを作成します。

ワークフロー定義ファイルについて：
- プロジェクトのルートディレクトリに`.github/workflows/[任意の名前].yml`を作成
- 拡張子は必ず`.yml`または`.yaml`

ワークフローの基本的な処理ステップ：
1. **コードのチェックアウト**: リポジトリからコードを取得
2. **Google Cloud認証**: GCPへのアクセス権限の設定
3. **Artifact Registryログイン**: コンテナイメージの保存先への認証
4. **コンテナイメージのビルドとプッシュ**: 
   - Dockerfileを使用してコンテナイメージをビルド
   - Artifact Registryにイメージをプッシュ
5. **Cloud Runへのデプロイ**: 
   - ビルドしたコンテナイメージをCloud Runにデプロイ
   - 環境変数やリソース設定の適用

以下はPython環境の例です：
```yaml
name: Deploy to Cloud Run

on:
  push:
    branches:
      - main  # メインブランチへのプッシュ時に実行

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }} # GitHubに登録したGCPのプロジェクトID
  SERVICE_NAME: python-app  # Cloud Runのサービス名
  REGION: asia-northeast1  # デプロイするリージョン（Artifact Registryのリポジトリと同じにする）
  REPOSITORY_NAME: my-app  # Artifact Registryのリポジトリ名

permissions:
  contents: 'read'  # リポジトリ内容の読み取り権限
  id-token: 'write' # OIDC認証トークンの生成権限

jobs:
  deploy:
    runs-on: ubuntu-latest  # 最新版のubuntuを指定

    steps:
    # リポジトリからコードを取得
    - name: Checkout code
      uses: actions/checkout@v4

    # Google Cloudへの認証を設定
    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v2
      with:
        credentials_json: ${{ secrets.GCP_SA_KEY }}
        project_id: ${{ env.PROJECT_ID }}

    # Artifact Registryへのログイン
    - name: 'Login to Artifact Registry'
      uses: 'docker/login-action@v3'
      with:
        registry: ${{ env.REGION }}-docker.pkg.dev
        username: _json_key
        password: ${{ secrets.GCP_SA_KEY }}
        
    # コンテナイメージのビルドとArtifact Registryへのプッシュ
    - name: Build and Push Container
      run: |-
        IMAGE_URI="${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPOSITORY_NAME }}/${{ env.SERVICE_NAME }}:${{ github.sha }}"
        docker build -t "$IMAGE_URI" .
        docker push "$IMAGE_URI"

    # Cloud Runへのデプロイ
    - name: Deploy to Cloud Run
      run: |-
        IMAGE_URI="${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPOSITORY_NAME }}/${{ env.SERVICE_NAME }}:${{ github.sha }}"
        gcloud run deploy ${{ env.SERVICE_NAME }} \
          --image "$IMAGE_URI" \
          --region ${{ env.REGION }} \
          --platform managed \
          --allow-unauthenticated \
          --set-secrets="KEY=SECRET_NAME:VERSION:latest" \
```

:::details ワークフローファイルの表記法：
**主要項目**
| セクション | 説明 | 用途 |
|:---|:---|:---|
| `name` | ワークフローの名前 | GitHubのActionsタブでの表示名 |
| `on` | トリガーイベントの設定 | プッシュ、PR、手動実行などの条件を指定 |
| `env` | 環境変数の定義 | プロジェクトID、リージョンなどの共通設定 |
| `jobs` | 実行するジョブの定義 | 並列実行可能な処理のグループ化 |
| `jobs.<job_id>` | ジョブの識別子（例: `deploy`） | 任意の名前を付けられる一意のジョブ識別子 |
| `jobs.<job_id>.runs-on` | 実行環境の指定 | Ubuntu、Windows、macOSなどから選択 |
| `jobs.<job_id>.steps` | 実行手順の定義 | 具体的なタスクの実行順序を指定 |
- `jobs.<job_id>`の`.`は階層構造を表します
- `<job_id>`は実際のYAMLファイルでは任意の名前（例: `deploy`）に置き換えます
- 例えば、以下のように記述します：
  ```yaml
  jobs:
    deploy:      # これが<job_id>の部分
      runs-on: ubuntu-latest
      steps:
        - name: First step
          run: echo "Hello"
  ```
  
**使用できる主な参照形式**
- `${{ secrets.SECRET_NAME }}`: GitHubのSecretsに登録した値
- `${{ github.* }}`: GitHubのコンテキスト（例：`github.repository`, `github.sha`）
- `${{ env.* }}`: 設定した環境変数
- `${{ vars.* }}`: GitHubに登録した変数
- `${{ inputs.* }}`: ワークフローの入力パラメータ
- `${{ needs.* }}`: 他のジョブの出力値
:::

::: message
注意点
- `--allow-unauthenticated`フラグは必要に応じて削除（認証が必要な場合）
- Secret Managerの環境変数は、`--set-secrets="KEY=SECRET_NAME:VERSION"`の形式で指定
  - `KEY`: アプリケーションで使用する環境変数名
  - `SECRET_NAME`: Secret Managerに登録したシークレットの名前
  - `VERSION`: シークレットのバージョン（通常は`latest`を使用）
:::

# デプロイ
ここまでを完了し、ローカルで開発したコードをGitHubリポジトリにプッシュすると、自動でCloud RunへのデプロイとコンテナイメージをArtifact Registryに保存する処理が実行されます。

以降はプッシュのたびに実行され、自動でCloud Runアプリが更新されるようになります。

デプロイの状況はGitHubリポジトリの「Actions」タブで確認できます。

# おわりに
本記事では、GitHub ActionsとGoogle Cloud Runを連携させたCI/CDパイプラインの構築方法について解説しました。この仕組みにより、以下のメリットが得られます：

- **コードのプッシュから本番環境へのデプロイまでを自動化**
- 手作業によるミスの削減
- **デプロイ頻度の向上**による開発サイクルの短縮
- チームメンバー間の環境差異の解消

CI/CDパイプラインは、最初の設定に少し手間がかかりますが、一度構築してしまえば**開発効率が大幅に向上**します。特にチーム開発においては、全員が同じ手順でデプロイできるため、**属人化を防ぐ効果**もあります。

今回は基本的な構成についてのみ紹介しましたが、実際のプロジェクトでは以下のような応用も検討してみてください：

- **テスト自動化**：ユニットテスト、統合テストの自動実行
- **複数環境対応**：開発環境、ステージング環境、本番環境の分離
- **通知機能**：Slack連携やメール通知によるデプロイ状況の共有

**CI/CDの導入は、近年のソフトウェア開発において必須のプラクティス**となっています。ぜひ本記事を参考に、自分のプロジェクトにCI/CDを導入してみてください。