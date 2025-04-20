# FastMCPについて
PythonでMCPサーバーを構築するための複雑な低レベル実装を抽象化して、**簡単かつ最小限のコードでMCPサーバーを構築できるようにしたフレームワーク**です。

開発当初は独立したプロジェクト(`jlowin/fastmcp`)でしたが、現在は公式のpython-SDKに統合されました。
https://github.com/modelcontextprotocol/python-sdk


## 動作環境
- **Python 3.8+**

## インストール
```bash
# uvを使う場合（推奨）
uv add "mcp[cli]"

# pipを使う場合
pip install "mcp[cli]"
```

# サーバー構築

## シンプルなサーバー例

入力されたテキストをそのまま返すechoサーバー：

```python
# echo_server.py
from mcp.server.fastmcp import FastMCP

# サーバーの作成
mcp = FastMCP("echoサーバー")

# 関数をツールとして登録
@mcp.tool()
def echo(text: str) -> str:
    """入力テキストをそのまま返す"""
    return text
```

## サーバーの実行

サーバーの実行コマンド：

```bash
# 開発モードで実行（MCP Inspectorでテストできます）
mcp dev echo_server.py

# 本番実行
mcp run echo_server.py
```

スクリプト内で直接実行する場合：
```python
if __name__ == "__main__":
    mcp.run()
```

## ポート・ホスト指定

インスタンス作成時に指定：

```python
mcp = FastMCP("My App", host="127.0.0.1", port=9000)
```

または、コマンドラインから：

```bash
mcp run echo_server.py --host 127.0.0.1 --port 9000
```

環境変数を使う場合：

```bash
FASTMCP_HOST=127.0.0.1 FASTMCP_PORT=9000 mcp run echo_server.py
```

## 通信方式の指定

FastMCPは複数の通信方式をサポートしています：

通信方式を指定：
```python
mcp.run(transport="stdio")  # 標準入出力（デフォルト）
mcp.run(transport="sse")    # Server-Sent Events
```

非同期関数として実行：
```python
await mcp.run_stdio_async()
await mcp.run_sse_async()
```

コマンドラインから指定：

```bash
mcp run echo_server.py --transport sse
```

## 環境変数と設定ファイル

`.env`ファイルで指定：

```
FASTMCP_HOST=0.0.0.0
FASTMCP_PORT=8080
FASTMCP_DEBUG=true
FASTMCP_LOG_LEVEL=DEBUG
```

使用可能な設定パラメータ：

- `host`: サーバーのホスト名（デフォルト: "0.0.0.0"）
- `port`: サーバーのポート番号（デフォルト: 8000）
- `debug`: デバッグモード（デフォルト: false）
- `log_level`: ログレベル（"DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"、デフォルト: "INFO"）
- `dependencies`: サーバー環境にインストールする依存関係のリスト

# 主要機能

FastMCPの提供する主要な機能タイプとしてツール・リソース・プロンプトあります。
FastMCPの機能抽象化により、デコレータを使用して関数を実行可能なツール・リソース・プロンプトとして登録できます。

## リソース（Resources）

リソースはLLMにデータを提供する機能です。REST APIのGETエンドポイント的に使用できます。

```python
# 静的リソース
@mcp.resource("config://app")
def get_config() -> dict:
    """アプリケーションの設定情報を提供"""
    return {"app_name": "テストアプリ", "version": "1.0"}

# 動的リソース（パラメータ付き）
@mcp.resource("users://{user_id}/profile")
def get_user_profile(user_id: str) -> dict:
    """指定されたユーザーのプロファイル情報を提供"""
    return {"id": user_id, "name": f"ユーザー {user_id}"}
```

::: details
- リソースURIにはパラメータを`{param_name}`形式で埋め込めます
- 関数引数、戻り値には**型アノテーションを明示する必要**があります
:::

AIは以下のようにリソースにアクセスできます：

```
（AIの表示内容例）
ユーザー情報を取得するために、リソース `users://john/profile` にアクセスします。
結果: "ユーザー john のプロファイル"
```

## ツール（Tools）

ツールはLLMが指示してMCPサーバー側でアクションを実行します。

```python
@mcp.tool()
def calculate_sum(a: int, b: int) -> int:
    """2つの数値の合計を計算"""
    return a + b

# 非同期ツール
@mcp.tool()
async def fetch_weather(city: str) -> str:
    """指定した都市の天気情報を取得"""
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://api.weather.com/{city}")
        return response.text
```

AIは以下のようにツールを呼び出せます：

```
（AIの表示内容例）
計算のために `calculate_sum` ツールを使用します。パラメータ: a=5, b=3
結果: 8
```

## プロンプト（Prompts）

プロンプトはLLMとの対話のためのテンプレート指定として使用します。

```python
@mcp.prompt()
def code_review(code: str) -> str:
    """コードレビュー用のプロンプト"""
    return f"以下のコードをレビューしてください：\n\n{code}"

# 構造化メッセージとして返す
from mcp.types import base

@mcp.prompt()
def error_analysis(error: str) -> list[base.Message]:
    """エラー分析プロンプト"""
    return [
        base.UserMessage("以下のエラーが発生しました："),
        base.UserMessage(error),
        base.AssistantMessage("エラーの原因を分析します。")
    ]
```
::: details
- プロンプト関数は単純な文字列を返すか、`base.UserMessage`と`base.AssistantMessage`を使って複数メッセージの配列を返せます
:::

AIは以下のようにプロンプトを使用できます：

```
（AIの表示内容例）
エラー分析のために `error_analysis` プロンプトを使います。
以下のエラーが発生しました：
TypeError: cannot concatenate 'str' and 'int' objects
エラーの原因を分析します。
```

# その他の機能

## コンテキスト（Context）

ツールやリソース内でコンテキストを利用するには、関数のパラメータに`Context`型の引数を追加します：

```python
from mcp.server.fastmcp import FastMCP, Context

@mcp.tool()
def process_with_context(data: str, ctx: Context) -> str:
    """コンテキストを利用して処理する"""
    # ログメッセージを送信
    ctx.info(f"処理開始: {data}")
    
    # 進捗報告
    ctx.report_progress(50, 100)
    
    # リソースへのアクセス
    config = ctx.read_resource("config://app")
    
    return f"処理完了: {data}"
```

::: details
- コンテキストは引数名が何であれ`ctx: Context`のように型アノテーションを付けるだけで自動的に注入されます
- コンテキストを通じて進捗報告（`report_progress`）、ログ出力（`debug`, `info`, `warning`, `error`）、リソースアクセス（`read_resource`）などが可能です
- リクエスト情報は`ctx.request_id`や`ctx.client_id`で取得できます
:::

### ログ出力機能

異なるレベルのログメッセージを出力：

```python
@mcp.tool()
async def log_messages(ctx: Context) -> str:
    """各種ログレベルのメッセージを出力する例"""
    # デバッグレベル（詳細な技術情報）
    await ctx.debug("デバッグ情報: データ処理開始")
    
    # 情報レベル（一般的な処理情報）
    await ctx.info("処理を開始しました")
    
    # 警告レベル（問題の可能性があるが処理は継続）
    await ctx.warning("ファイルが存在しませんが、デフォルト値を使用します")
    
    # エラーレベル（処理に問題があるが致命的ではない）
    await ctx.error("APIへの接続に失敗しました")
    
    return "ログ出力完了"
```

**出力例**：
```
[DEBUG] デバッグ情報: データ処理開始
[INFO] 処理を開始しました
[WARNING] ファイルが存在しませんが、デフォルト値を使用します
[ERROR] APIへの接続に失敗しました
```

### 進捗報告機能

長時間かかる処理の進捗状況をリアルタイムで報告：

```python
import asyncio

@mcp.tool()
async def long_running_task(ctx: Context) -> str:
    """時間のかかる処理で進捗を報告する例"""
    total_steps = 5
    
    for step in range(1, total_steps + 1):
        # 現在のステップと全体のステップ数を報告
        await ctx.report_progress(step, total_steps)
        
        # 何か時間のかかる処理を実行
        await asyncio.sleep(1)  # 例として1秒待機
        
        # 進捗とともにメッセージも報告できる
        await ctx.info(f"ステップ {step}/{total_steps} 完了")
    
    return "すべての処理が完了しました"
```

**出力例**：
```
[進捗] 1/5 (20%)
[INFO] ステップ 1/5 完了
[進捗] 2/5 (40%)
[INFO] ステップ 2/5 完了
[進捗] 3/5 (60%)
[INFO] ステップ 3/5 完了
[進捗] 4/5 (80%)
[INFO] ステップ 4/5 完了
[進捗] 5/5 (100%)
[INFO] ステップ 5/5 完了
```

### リソースアクセス機能

コンテキストを通じて他のリソースにアクセス：

```python
# リソースの定義
@mcp.resource("users://database")
def get_users() -> dict:
    """ユーザーデータベース"""
    return {
        "users": [
            {"id": 1, "name": "田中太郎"},
            {"id": 2, "name": "鈴木花子"}
        ]
    }

@mcp.resource("config://settings")
def get_settings() -> dict:
    """アプリケーション設定"""
    return {
        "max_users": 100,
        "timeout": 30
    }

# ツール内からリソースにアクセス
@mcp.tool()
async def access_resources(user_id: int, ctx: Context) -> dict:
    """リソースにアクセスする例"""
    # ユーザーデータベースを取得
    users_data = await ctx.read_resource("users://database")
    
    # 設定を取得
    settings = await ctx.read_resource("config://settings")
    
    # 指定されたIDのユーザーを探す
    user = next(
        (u for u in users_data["users"] if u["id"] == user_id), 
        None
    )
    
    if user:
        await ctx.info(f"ユーザー {user['name']} が見つかりました")
        return {
            "user": user,
            "settings": settings
        }
    else:
        await ctx.warning(f"ID {user_id} のユーザーは見つかりませんでした")
        return {"error": "ユーザーが見つかりません"}
```

**出力例** (user_id=1の場合):
```
[INFO] ユーザー 田中太郎 が見つかりました
{
  "user": {"id": 1, "name": "田中太郎"},
  "settings": {"max_users": 100, "timeout": 30}
}
```

### リクエスト情報へのアクセス

コンテキストからリクエストやセッションの情報にアクセス：

```python
@mcp.tool()
def get_request_info(ctx: Context) -> dict:
    """リクエスト情報を取得する例"""
    return {
        "request_id": ctx.request_id,
        "client_id": ctx.client_id or "不明",
        "session_data": {
            "type": str(type(ctx.session)),
            "has_session": ctx.session is not None
        }
    }
```

**出力例**:
```
{
  "request_id": "req_a1b2c3d4e5f6",
  "client_id": "client_123456",
  "session_data": {
    "type": "<class 'mcp.server.session.ServerSession'>",
    "has_session": true
  }
}
```

### ライフスパンコンテキスト

アプリケーション全体で共有されるステートを管理：

```python
from dataclasses import dataclass
from contextlib import asynccontextmanager
from collections.abc import AsyncIterator
import asyncio

# アプリケーションの状態を定義
@dataclass
class AppState:
    request_count: int = 0
    active_users: set = None
    
    def __post_init__(self):
        if self.active_users is None:
            self.active_users = set()

# ライフスパンコンテキストを設定
@asynccontextmanager
async def app_lifespan(server: FastMCP) -> AsyncIterator[AppState]:
    """アプリケーションのライフスパン管理"""
    # 初期化（サーバー起動時に実行）
    print("サーバーを起動中...")
    state = AppState()
    
    try:
        yield state  # アプリケーション実行中はこのステートが利用可能
    finally:
        # クリーンアップ（サーバー終了時に実行）
        print(f"サーバー終了: 合計 {state.request_count} リクエストを処理")

# ライフスパンコンテキストを使用するサーバー
mcp = FastMCP("ステート管理", lifespan=app_lifespan)

@mcp.tool()
async def track_user(user_id: str, ctx: Context) -> dict:
    """ユーザーのアクティビティを追跡"""
    # ライフスパンコンテキストにアクセス
    state = ctx.request_context.lifespan_context
    
    # リクエストカウンターをインクリメント
    state.request_count += 1
    
    # ユーザーを記録
    state.active_users.add(user_id)
    
    await ctx.info(f"アクティブユーザー数: {len(state.active_users)}")
    
    return {
        "request_number": state.request_count,
        "active_users": list(state.active_users),
        "user_count": len(state.active_users)
    }
```

**出力例** (複数回の呼び出し後):
```
[サーバー起動時] サーバーを起動中...

[1回目の呼び出し]
[INFO] アクティブユーザー数: 1
{
  "request_number": 1,
  "active_users": ["user1"],
  "user_count": 1
}

[2回目の呼び出し - 別のユーザー]
[INFO] アクティブユーザー数: 2
{
  "request_number": 2,
  "active_users": ["user1", "user2"],
  "user_count": 2
}

[サーバー終了時] サーバー終了: 合計 2 リクエストを処理
```

AIとユーザーには以下のように表示されます：

```
（AIの表示内容例）
`process_with_context` ツールを実行中...
[情報] 処理開始: テストデータ
進捗状況: 50/100
処理完了: テストデータ
```

## イメージ (Image)

LLMに画像を返すには`Image`クラスを使用します：

```python
from mcp.server.fastmcp import FastMCP, Image
from PIL import Image as PILImage

mcp = FastMCP("Image Server")

@mcp.tool()
def create_thumbnail(image_path: str) -> Image:
    """画像からサムネイルを作成"""
    img = PILImage.open(image_path)
    img.thumbnail((100, 100))
    return Image(data=img.tobytes(), format="png")
```

::: details
- `Image`クラスは戻り値の型として使用し、ファイルパス（`path="./image.jpg"`）またはバイトデータ（`data=bytes_data, format="png"`）から初期化できます
- PNG、JPEG、GIF、WebPなどの一般的な画像形式をサポートしています
:::

## ライフスパン管理

サーバーの起動時と終了時のリソース初期化とクリーンアップを管理できます：

```python
from contextlib import asynccontextmanager
from collections.abc import AsyncIterator
from dataclasses import dataclass
from mcp.server.fastmcp import FastMCP

# データベース接続の例
class Database:
    @classmethod
    async def connect(cls):
        # DB接続処理
        print("データベース接続中...")
        return cls()
        
    async def disconnect(self):
        # DB切断処理
        print("データベース切断中...")

@dataclass
class AppContext:
    db: Database

@asynccontextmanager
async def app_lifespan(server: FastMCP) -> AsyncIterator[AppContext]:
    """型安全なコンテキストでアプリケーションのライフサイクルを管理"""
    # 起動時に初期化
    db = await Database.connect()
    try:
        yield AppContext(db=db)
    finally:
        # シャットダウン時にクリーンアップ
        await db.disconnect()

# ライフスパンをサーバーに渡す
mcp = FastMCP("DB App", lifespan=app_lifespan)

@mcp.tool()
def query_db(ctx: Context) -> str:
    """ツール内でDBコンテキストを利用"""
    db = ctx.request_context.lifespan_context.db
    return "DBクエリ実行結果"
```

## 既存のサーバーへの統合

FastMCPサーバーを既存のASGIアプリケーションに統合できます：

```python
from starlette.applications import Starlette
from starlette.routing import Mount
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("My App")

# FastMCPサーバーを既存のASGIサーバーにマウント
app = Starlette(
    routes=[
        Mount('/mcp', app=mcp.sse_app()),
    ]
)
```

# トラブルシューティング

よくある問題と解決策：

- **サーバーが起動しない**: ポート競合がないか確認。別のポートを試す
- **リソースが見つからない**: URIパターンが正しいか確認
- **ツールが呼び出せない**: パラメータ型が正しいか確認
- **非同期関数でエラー**: すべての非同期関数に`async`/`await`が正しく使われているか確認