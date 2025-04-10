---
title: "FastMCPを理解する：AIモデルを外部機能と連携させるための革新的フレームワーク"
emoji: "🔌"
type: "tech"
topics: ["AI", "MCP", "Claude", "Python", "FastMCP"]
published: false
---

# FastMCPを理解する

皆さん、AIモデルを外部のツールやデータソースと連携させる方法について考えたことはありますか？今日は、そのための革新的なフレームワーク「FastMCP」について解説します。

## はじめに

大規模言語モデル（LLM）は驚異的な能力を持っていますが、最新データへのアクセスや特定の機能の実行には制限があります。Model Context Protocol（MCP）は、この制限を克服するために設計されたオープンプロトコルで、FastMCPはそのPython実装を簡素化するフレームワークです。

この記事では以下の内容を解説します：
- MCPの基本概念と目的
- FastMCPの特徴と利点
- サーバー、リソース、ツール、プロンプトの実装方法
- 実際の応用例とベストプラクティス

## MCPとFastMCPの概要

### MCPとは何か？

Model Context Protocol（MCP）は、LLMと外部システムを連携させるための標準化されたプロトコルです。これにより、AIアシスタントが様々なサービスと対話できるようになります。

MCPのアーキテクチャは以下のコンポーネントで構成されています：

- **ホスト**：ユーザーが対話するアプリケーション（Claude Desktop、IDE、カスタムエージェントなど）
- **クライアント**：ホストアプリケーション内に存在し、MCP接続を管理するコンポーネント
- **サーバー**：特定の機能を提供する軽量プログラム（リソース、ツール、プロンプトを公開）

MCPサーバーが提供する主な機能：

1. **リソース**：読み取り専用データ（ファイル内容、API応答、データベース情報など）
2. **ツール**：LLMが呼び出せる関数（計算実行、API呼び出し、ファイル操作など）
3. **プロンプト**：再利用可能なテンプレートやワークフロー

### FastMCPとは？

FastMCPは、PythonでMCPサーバーを構築するための高レベルなフレームワークです。その特徴は：

- **高速な開発**：最小限のコードでMCPサーバーを構築可能
- **Pythonic**：Pythonらしい直感的なインターフェース
- **シンプルさ**：低レベルプロトコルの詳細を抽象化

FastMCPは当初独立したプロジェクトでしたが、現在は公式のpython-SDKに統合され、推奨される高レベルインターフェースとして機能しています。

## FastMCPの基本実装

### サーバーのセットアップ

FastMCPサーバーは、`FastMCP`クラスのインスタンス化から始まります：

```python
from mcp.server.fastmcp import FastMCP

# サーバーインスタンスを作成し、名前を付ける
mcp = FastMCP("MyExampleServer")

# 依存パッケージを指定する場合
mcp = FastMCP("MyAppWithDeps", dependencies=["pandas", "numpy"])
```

この`mcp`オブジェクトがサーバーの中心的な役割を果たし、機能の登録やプロトコル準拠の処理を行います。

### リソースの定義

リソースはLLMへのコンテキスト情報提供のための読み取り専用データです。`@mcp.resource`デコレータを使って定義します：

```python
# 静的リソースの例
@mcp.resource("config://app")
def get_config() -> str:
    """アプリケーションの静的な設定データを返します"""
    return "App configuration data..."

# テンプレート化された動的リソースの例
@mcp.resource("greeting://{name}")
def get_greeting(name: str) -> str:
    """指定された名前に対するパーソナライズされた挨拶を返します"""
    return f"Hello, {name}!"
```

リソースURIには静的なものと、パラメータを含むテンプレート（`{}`で表現）の二種類があります。関数のdocstringはリソースの説明として使用されます。

### ツールの定義

ツールはLLMが呼び出せる関数で、計算や外部システムとの対話などを行います。`@mcp.tool()`デコレータで定義します：

```python
import httpx

# 簡単な計算ツールの例
@mcp.tool()
def add(a: int, b: int) -> int:
    """二つの数値を加算します"""
    return a + b

# 外部APIを呼び出す非同期ツールの例
@mcp.tool()
async def fetch_weather(city: str) -> str:
    """指定された都市の現在の天気を取得します"""
    api_endpoint = f"https://api.example-weather.com/?city={city}"
    async with httpx.AsyncClient() as client:
        response = await client.get(api_endpoint)
        return response.text
```

関数のパラメータとdocstringは、LLMがツールをどのように使用するかを理解するために重要です。型ヒントも適切に設定しましょう。

### プロンプトの定義

プロンプトは再利用可能な対話テンプレートで、`@mcp.prompt()`デコレータで定義します：

```python
from mcp.server.fastmcp.prompts import UserMessage, AssistantMessage

# 単純な文字列を返すプロンプトの例
@mcp.prompt()
def review_code(code: str) -> str:
    """指定されたコードのレビューを依頼するプロンプト"""
    return f"Please review this Python code:\n\n```python\n{code}\n```\nWhat are your suggestions?"

# 構造化されたメッセージリストを返すプロンプトの例
@mcp.prompt()
def debug_error(error: str) -> list[UserMessage | AssistantMessage]:
    """特定のエラーに関するデバッグ会話を開始するプロンプト"""
    return [
        UserMessage(f"I encountered the following error:\n{error}"),
        AssistantMessage("Okay, I can help with that. Could you provide more context about when and how this error occurred?")
    ]
```

## 高度な機能と活用方法

### Contextオブジェクトの利用

Contextオブジェクトは、ツールやリソース関数の実行中に様々な機能へのアクセスを提供します：

```python
from mcp.server.fastmcp import FastMCP, Context

mcp = FastMCP("ContextExample")

@mcp.tool()
async def process_file(file_uri: str, ctx: Context) -> str:
    # ログ記録
    ctx.info(f"Processing file: {file_uri}")
    
    # 進捗報告
    await ctx.report_progress(50, 100)
    
    # 別のリソースへのアクセス
    config = await ctx.read_resource("config://app")
    
    return f"Successfully processed {file_uri}"
```

主なContextの機能：

| メソッド | 説明 | 用途 |
|---------|------|------|
| `ctx.info/debug/warning/error()` | ログ記録 | 処理情報の記録 |
| `await ctx.report_progress()` | 進捗報告 | 長時間タスクの状況通知 |
| `await ctx.read_resource()` | リソース読取 | 他のリソースへのアクセス |

### 画像処理の対応

FastMCPは画像データの処理にも対応しています：

```python
from mcp.server.fastmcp import FastMCP, Image
from PIL import Image as PILImage

mcp = FastMCP("ImageTools")

@mcp.tool()
def create_thumbnail(image_path: str) -> Image:
    """画像からサムネイルを作成します"""
    img = PILImage.open(image_path)
    img.thumbnail((128, 128))
    import io
    img_byte_arr = io.BytesIO()
    img.save(img_byte_arr, format='PNG')
    return Image(data=img_byte_arr.getvalue(), format='png')
```

### サーバーの実行方法

FastMCPサーバーは以下の方法で実行できます：

```python
# スクリプト末尾に追加
if __name__ == "__main__":
    mcp.run()  # デフォルトでStdioトランスポートを使用
```

実行方法の種類：

1. **開発/テスト用**: `mcp dev server.py`（MCP Inspectorと連携）
2. **Claude Desktop統合**: `mcp install server.py`
3. **直接実行（Stdio）**: `python server.py`
4. **Webサーバー（SSE）**: ASGIサーバー経由で`mcp.sse_app()`を提供

## 実際の応用例

FastMCPを使った実際の応用例：

- データベースエクスプローラ
- 気象情報API連携
- ファイルシステムアクセス
- メッセージングアプリ（iMessage）連携
- タスク管理ツール連携
- 株価データ取得

## エラー処理とデバッグ

FastMCPでのエラー処理は標準的なPythonの例外処理に基づいています：

```python
@mcp.tool()
def divide(a: int, b: int) -> float:
    """二つの数値の除算を行います"""
    try:
        return a / b
    except ZeroDivisionError:
        # クリアなエラーメッセージを提供
        raise ValueError("除数にゼロは使用できません")
```

デバッグには「MCP Inspector」が役立ちます：
- `mcp inspector`コマンドで起動
- ツールやリソースのテスト
- ログの確認
- サーバーパフォーマンスの監視

## FastMCPモジュールの詳細分析

FastMCPモジュールは`src/mcp/server/fastmcp/__init__.py`で定義された3つの主要な公開シンボルを提供しており、これらはMCPサーバー開発の基盤となります。

| シンボル | 説明 | 主な機能 |
|---------|------|---------|
| `FastMCP` | MCPサーバーを構築し実行するための主要なインターフェース | • `server.tool()`、`server.resource()`デコレータによるツール/リソースの登録/管理<br>• 事前定義したプロンプトの管理<br>• `run()`メソッドによるサーバー起動（stdio または SSE 通信方式） |
| `Context` | MCP機能にアクセスするためのインターフェース | • ログ機能: `debug()`, `info()`, `warning()`, `error()` メソッドでクライアントにログメッセージを送信<br>• 進捗報告: `report_progress()` メソッドで処理の進捗状況を報告<br>• リソースアクセス: `read_resource()` メソッドでリソースの内容を読取<br>• リクエスト情報: `client_id`, `request_id` などのプロパティでリクエスト情報にアクセス |
| `Image` | ツールから画像を返すためのヘルパークラス | • 多様な初期化: ファイルパスまたはバイナリデータから画像を作成<br>• MIME型自動判定: ファイル拡張子からMIME型を自動的に判定<br>• MCP互換: MCPプロトコルと互換性のある形式に変換するメソッドを提供 |

これらのコアシンボルを理解することで、MCPサーバーの構築と管理が効率的になります。特に`Context`オブジェクトはツールやリソース関数内での機能拡張に重要な役割を果たします。

## まとめ

FastMCPは、MCPプロトコルを活用してAIモデルと外部システムを簡単に連携させるためのPythonフレームワークです。その主な利点は：

- **開発の簡素化**：少ないコードで強力な連携機能を実現
- **Pythonic**：Pythonらしい直感的なインターフェース
- **柔軟性**：様々なシステムとの統合を容易に実現

今後、AIモデルの活用範囲が広がるにつれ、MCPとFastMCPの重要性はさらに高まると予想されます。ぜひ自分のプロジェクトでFastMCPを試してみてください！

## 参考リソース

- [MCP公式リポジトリ](https://github.com/anthropics/anthropic-cookbook/tree/main/mcp)
- [FastMCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [Claude Desktop](https://claude.ai/desktop) 



FastMCPについて
Python-SDKのFastMCPはMCPサーバーを構築する際の低レベルで複雑な実装を高レベルに抽象化し簡単に扱えるようにしたインターフェースで扱えるようにしたフレームワークです。
開発当初は独立したプロジェクト()でしたが、現在は公式のpython-SDKに統合され推奨されています。

このページは
FastMCPが抽象化している低レベルの実装を理解し、FastMCPの提供する機能や、MCPの詳細な実装について理解することを目的としています。

FastMCPが抽象化する機能・概念
FastMCPは主に以下の機能を抽象化してユーザーへ提供しています。
クラス　機能	API (mcp.server.fastmcp)	抽象化された低レベルの概念
`FastMCP`
...　サーバーインスタンス化	FastMCP(...)	トランスポート層のバインディング（stdio/SSE）、非同期イベントループ管理、サーバーライフサイクル管理、MCP初期化メッセージ処理
...　ツール定義	@mcp.tool()	関数登録、パラメータ検証（JSONスキーマ生成）、リクエストルーティング、非同期実行管理、コンテキスト注入
...　リソース定義	@mcp.resource()	関数登録、URIテンプレートマッチングとパラメータ抽出、リクエストルーティング、レスポンスシリアライゼーション（MIMEタイプ含む）、リソーステンプレートリスト処理、非同期実行管理
...　プロンプト定義	@mcp.prompt()	テンプレート登録、フォーマット処理
`Context`
`Image`

各機能の使い方は別記事を参照してください
記事URL

ツールの保存処理
@server.tool()デコレータでデコレーションされた関数は、ToolManagerクラスで引数をJSONスキーマに変換した後_tools辞書に保存されます


