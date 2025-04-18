# はじめに

この記事では、FastMCP (MCP Python SDK) の基本的な使い方を解説します。記事の内容は初心者から実務レベルまで、Python開発者がFastMCPを使って実際のMCPサーバーを開発・運用するのに役立つ情報を提供します。

## 想定読者

- Pythonでプログラミングの経験がある方
- MCPの基本概念を理解している方（[MCPとは？](リンク予定) 参照）
- AIアプリケーションをより強力にするためのMCPサーバーを構築したい方

## この記事のゴール

- 最小限のコードでFastMCP サーバーを立ち上げる方法の理解
- サーバー設定（ポート、ホスト、通信方式）の変更方法の習得
- 基本的な機能（ツール、リソース、プロンプト）の実装方法の理解
- 実際の使用例と動作確認方法の把握

# 環境準備

## インストール方法

FastMCPは公式のMCP Python SDKとして提供されています。以下のコマンドでインストールできます：

```bash
# uvを使う場合（推奨）
uv add "mcp[cli]"

# pipを使う場合
pip install "mcp[cli]"
```

## 前提知識

- **Python 3.8+**: FastMCPは3.8以上のPythonバージョンが必要です
- **非同期処理**: FastMCPはasync/awaitをサポートしていますが、同期関数も使用可能
- **ASGI**: Starlette/ASGIフレームワークを使っていますが、詳細を知らなくても使用可能

# サーバーの基本起動

## 最小限のエコーサーバー実装

FastMCPで最もシンプルなサーバーを作成してみましょう。入力されたテキストをそのまま返すエコーサーバーです：

```python
# echo_server.py
from mcp.server.fastmcp import FastMCP

# サーバーの作成
mcp = FastMCP("エコーサーバー")

@mcp.tool()
def echo(text: str) -> str:
    """入力テキストをそのまま返す"""
    return text
```

## サーバーの実行

サーバーを実行するには、以下のコマンドを使用します：

```bash
# 開発モードで実行（MCP Inspectorでテストできます）
mcp dev echo_server.py

# 本番実行
mcp run echo_server.py

# または、スクリプト内で直接実行する場合
# echo_server.pyファイルの末尾に以下を追加
if __name__ == "__main__":
    mcp.run()
```

# サーバー設定パラメータ

## ポート・ホスト指定

サーバーのホストとポートを指定するには、複数の方法があります：

```python
# FastMCPインスタンス作成時に指定
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

```python
# 通信方式を指定して直接実行
mcp.run(transport="stdio")  # 標準入出力（デフォルト）
mcp.run(transport="sse")    # Server-Sent Events

# または非同期関数として実行
await mcp.run_stdio_async()
await mcp.run_sse_async()
```

コマンドラインから：

```bash
mcp run echo_server.py --transport sse
```

## 環境変数と設定ファイル

FastMCPは`.env`ファイルと環境変数をサポートしています：

```
# .envファイルの例
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

# 機能登録

FastMCPは3つの主要な機能タイプを提供します。

## リソース（Resources）

リソースはLLMにデータを提供する機能です。REST APIのGETエンドポイントに類似しています。

```python
# 静的リソース
@mcp.resource("config://app")
def get_config() -> str:
    """アプリケーションの設定情報を提供"""
    return "アプリケーション設定データ"

# 動的リソース（パラメータ付き）
@mcp.resource("users://{user_id}/profile")
def get_user_profile(user_id: str) -> str:
    """指定されたユーザーのプロファイル情報を提供"""
    return f"ユーザー {user_id} のプロファイル"
```

AIは以下のようにリソースにアクセスできます：

```
（AIの表示内容例）
ユーザー情報を取得するために、リソース `users://john/profile` にアクセスします。
結果: "ユーザー john のプロファイル"
```

## ツール（Tools）

ツールはLLMがアクションを実行するための機能です。計算や副作用を持つ操作ができます。

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

プロンプトはLLMとの対話のためのテンプレートです。

```python
@mcp.prompt()
def code_review(code: str) -> str:
    """コードレビュー用のプロンプト"""
    return f"以下のコードをレビューしてください：\n\n{code}"

# 構造化メッセージとして返す
from mcp.types import base

@mcp.prompt()
def error_analysis(error: str) -> list[base.Message]:
    return [
        base.UserMessage("以下のエラーが発生しました："),
        base.UserMessage(error),
        base.AssistantMessage("エラーの原因を分析します。")
    ]
```

AIは以下のようにプロンプトを使用できます：

```
（AIの表示内容例）
エラー分析のために `error_analysis` プロンプトを使います。
以下のエラーが発生しました：
TypeError: cannot concatenate 'str' and 'int' objects
エラーの原因を分析します。
```

## コンテキスト（Context）の利用

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

AIとユーザーには以下のように表示されます：

```
（AIの表示内容例）
`process_with_context` ツールを実行中...
[情報] 処理開始: テストデータ
進捗状況: 50/100
処理完了: テストデータ
```

# AIとの対話例

以下はFastMCPサーバーをAIアシスタントに接続したときの対話例です：

# エコーサーバーの利用例

```
ユーザー: こんにちは、エコーサーバーが動いていますか？

AI: こんにちは！エコーサーバーが動作しているか確認してみますね。

「こんにちは、動作確認中です」というテキストをエコーします。
結果: こんにちは、動作確認中です

はい、エコーサーバーは正常に動作しています。
```

## 計算ツールの利用例

```
ユーザー: 125と48を足した結果を教えてください

AI: 125と48の合計を計算します。

計算ツール `calculate_sum` を使用します。パラメータ: a=125, b=48
結果: 173

125と48の合計は173です。
```

## リソースアクセスの例

```
ユーザー: ユーザーIDが12345のプロファイル情報を教えて

AI: ユーザーID 12345のプロファイル情報を取得します。

リソース `users://12345/profile` にアクセスします。
結果: "ユーザー 12345 のプロファイル"

こちらがユーザー12345のプロファイル情報です。
```

# 応用例

## 画像サポート

FastMCPは`Image`クラスを使って画像データを扱えます：

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

# まとめ

FastMCP (MCP Python SDK) は、最小限のコードでMCPサーバーを構築するための強力で使いやすいフレームワークです。この記事で紹介した基本的な使い方を理解すれば、すぐに実用的なMCPサーバーを構築し、AIアプリケーションと連携させることができます。

サーバーの設定、機能の登録、AIとの対話方法など、実践的な内容を中心に解説しました。より詳しい情報は[公式ドキュメント](https://modelcontextprotocol.io)を参照してください。

FastMCPを活用して、AIアプリケーションをより強力かつインテリジェントにしましょう！