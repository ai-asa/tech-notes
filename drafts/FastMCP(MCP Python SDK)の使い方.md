---
実践的な内容にする。FastMCPの使い方を知りたいユーザーがメインターゲット
機能の詳細な解説というより、提供機能の目的と使い方、実装例を紹介する。マニュアル的な記事
---

この記事はFastMCP (MCP Python SDK) 関連の記事です。

MCPとは？APIとの違い
FastMCP (MCP Python SDK) の使い方 -> URL埋め込みなし
Python標準ライブラリでMCPサーバーを作ってみる -> URL埋め込み
FastMCP (MCP Python SDK) の実装と抽象化を理解する -> URL埋め込み
  
# FastMCP (MCP Python SDK) について
PythonでMCPサーバーを構築するための複雑な低レベル実装を抽象化して、**簡単かつ最小限のコードでMCPサーバーを構築できるようにしたフレームワーク**です。

開発当初は独立したプロジェクト(`jlowin/fastmcp`)でしたが、現在は公式のpython-SDKに統合されました。ライセンスもApache2.0 -> MITへ変更
https://github.com/modelcontextprotocol/python-sdk

# FastMCPの機能
## 提供機能一覧

以下の表は、FastMCPが提供する主要な機能と、その目的をまとめたものです。

| 機能 | 目的 | 具体的な使用例 |
|------|------|--------|
| リソース (Resources) | LLMにデータを提供する<br>REST APIのGETエンドポイントに相当 | • データベースからユーザー情報を読み取り<br>• 社内ドキュメントやナレッジベースをLLMに提供<br>• 設定ファイルや環境情報の取得 |
| ツール (Tools) | LLMがアクションを実行するための機能<br>計算や副作用を持つ操作 | • ユーザーの入力に基づく計算処理<br>• 外部APIの呼び出し（天気情報取得など）<br>• データベースへの書き込み処理 |
| プロンプト (Prompts) | LLMに特定のタスクを実行させるための<br>定型メッセージやテンプレート | • 「このコードをレビューして」といった指示を定型化<br>• 質問-回答の流れを構造化された対話として定義<br>• LLMに特定フォーマットでの回答を促すガイド |
| コンテキスト (Context) | ツールやリソース内からMCP機能へ<br>アクセスするためのオブジェクト | • 長時間処理の進捗状況をリアルタイム報告<br>• LLMとユーザーへのログメッセージ送信<br>• ツール内から別のリソースにアクセス |
| 画像サポート (Images) | 画像データの処理と返却を簡単に行う機能 | • アップロードされた画像からサムネイル生成<br>• データを可視化したグラフやチャートの生成<br>• 画像データの一貫した形式での返却 |
| ライフスパン管理 | サーバー起動時と終了時の<br>リソース初期化・クリーンアップ | • データベース接続の確立と安全な切断<br>• APIキーなどの認証情報の初期化<br>• 一時ファイルの作成と削除 |

## 抽象化

FastMCPはMCPプロトコルの低レベル実装を抽象化することで、開発者がMCPサーバーを簡単に構築できるようにしています。以下は主な抽象化の一覧です：

| 抽象化機能 | 説明 | 代替実装との比較 |
|----------|------|----------------|
| デコレータベースのAPI | `@mcp.tool()`, `@mcp.resource()`, `@mcp.prompt()`などのデコレータで関数を簡単に登録 | 低レベル実装ではハンドラー関数を手動で登録する必要がある |
| 型ヒントによる自動バリデーション | Pydanticの型システムを活用し、関数パラメータの検証と型変換を自動化 | 手動でのパラメータ検証と型変換、エラー処理が必要 |
| 非同期処理の自動検出 | 同期関数と非同期関数を区別せず、適切に実行 | 非同期処理を手動で管理する必要がある |
| コンテキスト注入 | 関数シグネチャに基づき、コンテキストオブジェクトを自動的に注入 | コンテキストの手動での受け渡しが必要 |
| ライフスパン管理 | サーバーの起動/終了時の処理と共有リソースを簡単に管理 | 複雑なライフサイクルフックの実装が必要 |
| プロトコル互換性の自動管理 | MCPプロトコルのバージョン互換性やメッセージフォーマットを自動処理 | プロトコル仕様への対応を手動で実装する必要がある |

# FastMCPの使い方
## インストール方法

```bash
# uvを使う場合（推奨）
uv add "mcp[cli]"

# pipを使う場合
pip install "mcp[cli]"
```

## MCPサーバーへの関数の登録方法

FastMCPでは、デコレータを使って簡単に関数をMCPサーバーに登録できます。以下は基本的な登録パターンです：

### ツールの登録

ツールはLLMがアクションを実行するための関数です。`@mcp.tool()`デコレータを使用して登録します：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("My App")

@mcp.tool()
def calculate_sum(a: int, b: int) -> int:
    """2つの数値の合計を計算"""
    return a + b

# 名前と説明を明示的に指定
@mcp.tool(name="multiply", description="2つの数値の積を計算")
def calc_product(x: int, y: int) -> int:
    return x * y
```

### リソースの登録

リソースはLLMにデータを提供する関数です。`@mcp.resource()`デコレータを使って登録します：

```python
# 静的リソース
@mcp.resource("config://app")
def get_app_config() -> str:
    """アプリケーションの設定情報を提供"""
    return "アプリケーション設定データ"

# 動的リソース（パラメータ付き）
@mcp.resource("users://{user_id}/profile")
def get_user_profile(user_id: str) -> str:
    """指定されたユーザーのプロファイル情報を提供"""
    return f"ユーザー {user_id} のプロファイル"
```

### プロンプトの登録

プロンプトはLLMとの対話のためのテンプレートです。`@mcp.prompt()`デコレータを使って登録します：

```python
@mcp.prompt()
def code_review(code: str) -> str:
    """コードレビュー用のプロンプト"""
    return f"以下のコードをレビューしてください：\n\n{code}"

# 構造化メッセージとして返す
@mcp.prompt()
def error_analysis(error: str) -> list[base.Message]:
    return [
        base.UserMessage("以下のエラーが発生しました："),
        base.UserMessage(error),
        base.AssistantMessage("エラーの原因を分析します。")
    ]
```

### コンテキストの利用

関数内でコンテキストを利用するには、関数のパラメータに`Context`型の引数を追加します：

```python
from mcp.server.fastmcp import FastMCP, Context

@mcp.tool()
def process_with_context(data: str, ctx: Context) -> str:
    """コンテキストを利用して処理する"""
    # ログメッセージ
    ctx.info(f"処理開始: {data}")
    
    # 進捗報告
    ctx.report_progress(50, 100)
    
    # リソースへのアクセス
    config = ctx.read_resource("config://app")
    
    return f"処理完了: {data}"
```

## 提供機能

FastMCPは以下の主要な機能を提供しています：

### リソース (Resources)

リソースはLLMにデータを提供するための機能です。REST APIのGETエンドポイントのように、データを提供するだけで、大きな計算や副作用を持つべきではありません。

```python
@mcp.resource("config://app")
def get_config() -> str:
    """静的な設定データを提供"""
    return "アプリの設定情報"

@mcp.resource("users://{user_id}/profile")
def get_user_profile(user_id: str) -> str:
    """ユーザーIDに基づく動的なデータ"""
    return f"ユーザー {user_id} のプロファイルデータ"
```

### ツール (Tools)

ツールはLLMがアクションを実行するための機能です。リソースとは異なり、計算を実行し副作用を持つことが期待されています。

```python
@mcp.tool()
def calculate_bmi(weight_kg: float, height_m: float) -> float:
    """体重と身長からBMIを計算"""
    return weight_kg / (height_m**2)

@mcp.tool()
async def fetch_weather(city: str) -> str:
    """指定した都市の天気情報を取得"""
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://api.weather.com/{city}")
        return response.text
```

### プロンプト (Prompts)

プロンプトは、LLMがサーバーと効果的に対話するための再利用可能なテンプレートです。

```python
@mcp.prompt()
def review_code(code: str) -> str:
    return f"以下のコードをレビューしてください：\n\n{code}"

@mcp.prompt()
def debug_error(error: str) -> list[base.Message]:
    return [
        base.UserMessage("以下のエラーが発生しています："),
        base.UserMessage(error),
        base.AssistantMessage("デバッグを手伝います。今までに試したことは何ですか？"),
    ]
```

### コンテキスト (Context)

コンテキストオブジェクトは、ツールやリソースにMCPの機能へのアクセスを提供します：

```python
@mcp.tool()
async def long_task(files: list[str], ctx: Context) -> str:
    """複数のファイルを処理し、進捗状況を追跡"""
    for i, file in enumerate(files):
        ctx.info(f"{file}を処理中")
        await ctx.report_progress(i, len(files))
        data, mime_type = await ctx.read_resource(f"file://{file}")
    return "処理完了"
```

### 画像サポート (Images)

FastMCPは`Image`クラスを提供し、画像データを自動的に処理します：

```python
@mcp.tool()
def create_thumbnail(image_path: str) -> Image:
    """画像からサムネイルを作成"""
    img = PILImage.open(image_path)
    img.thumbnail((100, 100))
    return Image(data=img.tobytes(), format="png")
```

### ライフスパン管理

サーバーの起動時と終了時のリソース初期化とクリーンアップを管理できます：

```python
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
mcp = FastMCP("My App", lifespan=app_lifespan)
```

## 実装例

### 基本的なエコーサーバー

最もシンプルな例として、入力されたテキストをそのまま返すエコーサーバーの実装です：

```python
from mcp.server.fastmcp import FastMCP

# サーバーを作成
mcp = FastMCP("Echo Server")

@mcp.tool()
def echo(text: str) -> str:
    """入力テキストをそのまま返す"""
    return text
```

### パラメータ説明付きのサーバー

Pydanticの`Field`を使ってパラメータに詳細な説明を追加できます：

```python
from pydantic import Field
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Parameter Descriptions Server")

@mcp.tool()
def greet_user(
    name: str = Field(description="挨拶する相手の名前"),
    title: str = Field(description="Mr/Ms/Drなどの敬称（任意）", default=""),
    times: int = Field(description="挨拶を繰り返す回数", default=1),
) -> str:
    """任意の敬称と繰り返し回数でユーザーに挨拶する"""
    greeting = f"こんにちは、{title + ' ' if title else ''}{name}さん！"
    return "\n".join([greeting] * times)
```

### 複雑な入力を受け付けるサーバー

Pydanticモデルを使って複雑な入力の検証と処理ができます：

```python
from typing import Annotated
from pydantic import BaseModel, Field
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Shrimp Tank")

class ShrimpTank(BaseModel):
    class Shrimp(BaseModel):
        name: Annotated[str, Field(max_length=10)]

    shrimp: list[Shrimp]

@mcp.tool()
def name_shrimp(
    tank: ShrimpTank,
    # 関数シグネチャでもpydantic Fieldを使用して検証できます
    extra_names: Annotated[list[str], Field(max_length=10)],
) -> list[str]:
    """タンク内のすべてのエビの名前をリスト化"""
    return [shrimp.name for shrimp in tank.shrimp] + extra_names
```

### データベース統合の例

SQLiteデータベースとの統合例：

```python
import sqlite3
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("SQLite Explorer")

@mcp.resource("schema://main")
def get_schema() -> str:
    """データベーススキーマをリソースとして提供"""
    conn = sqlite3.connect("database.db")
    schema = conn.execute("SELECT sql FROM sqlite_master WHERE type='table'").fetchall()
    return "\n".join(sql[0] for sql in schema if sql[0])

@mcp.tool()
def query_data(sql: str) -> str:
    """SQLクエリを安全に実行"""
    conn = sqlite3.connect("database.db")
    try:
        result = conn.execute(sql).fetchall()
        return "\n".join(str(row) for row in result)
    except Exception as e:
        return f"エラー: {str(e)}"
```

## サーバーの実行方法

### 開発モード

MCPインスペクターでサーバーをテストおよびデバッグする最速の方法：

```bash
# 基本的な実行
mcp dev server.py

# 依存関係を追加
mcp dev server.py --with pandas --with numpy

# ローカルコードをマウント
mcp dev server.py --with-editable .
```

### Claude Desktop 統合

サーバーの準備ができたら、Claude Desktopにインストール：

```bash
# 基本的なインストール
mcp install server.py

# カスタム名
mcp install server.py --name "My Analytics Server"

# 環境変数
mcp install server.py -v API_KEY=abc123 -v DB_URL=postgres://...
mcp install server.py -f .env
```

### 直接実行

高度なシナリオのためのカスタムデプロイメント：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("My App")

if __name__ == "__main__":
    mcp.run()
```

実行方法：
```bash
python server.py
# または
mcp run server.py
```

### 既存のASGIサーバーへのマウント

SSEサーバーを既存のASGIサーバーにマウントして、他のASGIアプリケーションと統合できます：

```python
from starlette.applications import Starlette
from starlette.routing import Mount, Host
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("My App")

# SSEサーバーを既存のASGIサーバーにマウント
app = Starlette(
    routes=[
        Mount('/', app=mcp.sse_app()),
    ]
)

# または動的にホストとしてマウント
app.router.routes.append(Host('mcp.acme.corp', app=mcp.sse_app()))
```

## まとめ

FastMCP（MCP Python SDK）は、最小限のコードでMCPサーバーを構築するための強力なフレームワークです。リソース、ツール、プロンプト、コンテキスト、画像サポートなど、多くの機能を提供しており、LLMアプリケーションに対して構造化されたコンテキストと機能を提供するための優れた方法となっています。

シンプルなエコーサーバーから複雑なメモリシステムまで、さまざまなユースケースに対応でき、既存のASGIアプリケーションにも簡単に統合できます。

FastMCPを活用することで、LLMとの対話を強化し、よりインテリジェントで有用なアプリケーションを構築しましょう。

