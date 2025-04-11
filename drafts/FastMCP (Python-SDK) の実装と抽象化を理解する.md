# はじめに
## 誰に向けたものです？
- FastMCPの実装をある程度理解しておきたい
- FastMCPが抽象化している概念を理解したい
- **むしろFastMCPを使わずにMCPを一から実装したい**

FastMCPの使い方が知りたい方は以下の記事をどうぞ
https://qiita.com/oohira/items/35e6eaaf4b44613ad7d3
  
## MCPについて
**Model Context Protocol（MCP）** は、LLMと外部システムを連携させるためにAnthoropic社によって標準化されたプロトコルです。AIアシスタントが様々なサービスと対話できる環境を構築します。

MCPサーバーは主に以下の機能を提供します：
1. **リソース**：読み取り専用データ（ファイル内容、API応答、データベース情報など）
2. **ツール**：LLMが呼び出せる関数（計算実行、API呼び出し、ファイル操作など）
3. **プロンプト**：再利用可能なテンプレートやワークフロー

## FastMCPについて
PythonでMCPサーバーを構築するための複雑な低レベル実装を抽象化して、**簡単かつ最小限のコードでMCPサーバーを構築できるようにしたフレームワーク**です。

開発当初は独立したプロジェクト(`jlowin/fastmcp`)でしたが、現在は公式のpython-SDKに統合されました。ライセンスもApache2.0 -> MITへ変更
https://github.com/modelcontextprotocol/python-sdk

# MCPの実装について
## 前提となる知識
### MPCの通信プロトコル
MCP の通信は、JSON-RPC 2.0 プロトコルに基づいています。クライアントとサーバ間の通信は、標準入出力（stdio）または HTTP with Server-Sent Events（SSE）トランスポートを通じて行われます。

JSON-RPC 2.0は、リモートプロシージャコールを実現するためのシンプルなプロトコルです。基本的なリクエストとレスポンスの構造は以下のようになっています：

```json
// リクエスト
{
  "jsonrpc": "2.0",
  "method": "call_tool",
  "params": {
    "name": "add",
    "arguments": {"a": 5, "b": 3}
  },
  "id": 123
}

// レスポンス
{
  "jsonrpc": "2.0",
  "result": {"content": "8", "mime_type": "text/plain"},
  "id": 123
}
```

MCPでは、これらのJSON-RPCメッセージが`types.JSONRPCMessage`クラスとしてシリアライズ/デシリアライズされます。

### stdioトランスポート
標準入出力（stdio）トランスポートは最もシンプルなMCP接続方法で、1行ごとのJSON-RPCメッセージのやり取りで実装されています。以下は実際の動作フロー：

1. クライアントは標準入力を通じてJSON-RPCリクエストを送信（1行に1リクエスト）
2. サーバーは標準出力にJSON-RPCレスポンスを書き込む（1行に1レスポンス）
3. すべての通信はUTF-8エンコードされた文字列として扱われる

MCPのsdtioトランスポートは`mcp.server.stdio`モジュールで実装されており、任意のstdinとstdoutストリームをラップし、非同期オブジェクトストリームに変換します。

### SSE
Server-Sent Events（SSE）は、サーバーからクライアントへの一方向のリアルタイム通信を可能にするHTTPベースの技術です。MCPの場合：

1. クライアントはサーバーにHTTP GET接続を確立し、`text/event-stream`コンテンツタイプを受信
2. サーバーはこの接続を通じて非同期にメッセージを送信
3. クライアントからサーバーへのメッセージは別のHTTP POSTリクエストで送信

MCPのSSEトランスポートは`mcp.server.sse`モジュールの`SseServerTransport`クラスで実装されており、以下の機能を提供：

1. `connect_sse`メソッド：SSEのHTTP接続を処理し、クライアントに一意のセッションIDを発行
2. `handle_post_message`メソッド：クライアントからのHTTP POSTリクエストを処理し、メッセージをセッションに関連付ける

### ASGI
ASGI（Asynchronous Server Gateway Interface）は、非同期WebアプリケーションのためのPythonの標準インターフェースです。MCPのSSEトランスポートはASGIアプリケーションとして実装されており、StarlettやUvicornなどのASGIサーバーで簡単に実行できます。

ASGIインターフェースは`app(scope, receive, send)`シグネチャを持つ関数として実装され、MCPサーバーはこのインターフェースを使用してHTTPエンドポイントを公開します。FastMCPはStarletteフレームワークを使用して、ASGIエンドポイントを簡単に作成できるようにしています。

## MCPの主要な実装
### 構成
MCPを実装するためには、以下の主要コンポーネントが必要です：

1. **トランスポートレイヤー**：JSON-RPC通信を処理（stdio/SSE）
2. **セッション管理**：クライアントとの接続を管理
3. **要求処理システム**：JSON-RPCリクエストを受け取り、適切なハンドラーに転送
4. **ハンドラー登録**：ツール、リソース、プロンプトなどの機能を登録
5. **コンテキスト管理**：リクエスト間の状態と情報を管理
6. **スキーマ生成**：型からJSON Schemaを自動生成

python-SDKでは、これらの機能が以下のモジュールに分かれています：

- `mcp.types`：MCPのデータ型と変換機能
- `mcp.server.lowlevel`：低レベルなMCPサーバー実装
- `mcp.server.stdio`/`mcp.server.sse`：トランスポート実装
- `mcp.server.session`：セッション管理
- `mcp.server.fastmcp`：高レベルな抽象化API

### 簡単な実装
FastMCPを使わずに、低レベルのAPIを使用してMCPサーバーを実装する最小限のコードは以下のようになります：

```python
import asyncio
from mcp.server.lowlevel.server import Server
import mcp.types as types
from mcp.server.stdio import stdio_server
from mcp.server.models import InitializationOptions

# サーバーインスタンスの作成
server = Server("My Low-Level MCP Server")

# リクエストハンドラーの登録
@server.list_tools()
async def handle_list_tools():
    """利用可能なツールの一覧を返す"""
    return [
        types.Tool(
            name="add",
            description="Add two numbers",
            inputSchema={"type": "object", "properties": {
                "a": {"type": "number"},
                "b": {"type": "number"}
            }, "required": ["a", "b"]}
        )
    ]

@server.call_tool()
async def handle_call_tool(name: str, arguments: dict):
    """ツールを呼び出す"""
    if name == "add":
        result = arguments["a"] + arguments["b"]
        return [types.TextContent(content=str(result))]
    raise ValueError(f"Unknown tool: {name}")

@server.list_resources()
async def handle_list_resources():
    """利用可能なリソースの一覧を返す"""
    return [
        types.Resource(
            uri="hello://world",
            name="Hello World",
            description="A simple greeting",
            mimeType="text/plain"
        )
    ]

@server.read_resource()
async def handle_read_resource(uri):
    """リソースの内容を読み取る"""
    if uri == "hello://world":
        return "Hello, World!"
    raise ValueError(f"Unknown resource: {uri}")

# サーバーの実行
async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options()
        )

if __name__ == "__main__":
    asyncio.run(main())
```

この低レベル実装では、各ハンドラーが明示的に登録され、型の変換や検証も手動で行う必要があります。FastMCPはこれらを大幅に簡素化します。

# FastMCPの実装と抽象化

## FastMCPが抽象化する主要な機能

FastMCPは、MCPサーバー実装の複雑さを隠蔽し、開発者が本質的な機能に集中できるよう、以下の主要な機能を抽象化しています：

1. **インターフェース実装**
   - デコレータパターンによる宣言的API設計

2. **プロトコル処理**
   - 型ヒントからJSON-RPC/MCPスキーマへの自動変換
   - 非同期コミュニケーションの管理
   - 複数トランスポート（stdio/SSE）の抽象化

3. **基盤機能**
   - コンテキスト機能の自動注入
   - URIベースのリソースシステム
   - ライフサイクル管理

## FastMCPの実装アプローチ

FastMCPは、Pythonの高度な機能を活用してMCPの複雑さを抽象化しています。特に関数デコレータ、型ヒント解析、asyncioベースの非同期処理、コンテキストパターンなどを組み合わせることで、開発者が簡潔で直感的なコードを書けるようにしています。

## 各抽象化機能の実装詳細

### デコレータパターンによる宣言的API

FastMCPは、Pythonのデコレータ機能を活用して、開発者が単純なデコレータを関数に付与するだけでMCP機能として登録できる宣言的なAPIを提供しています。

例えば、以下のようにして関数をツールとして登録できます：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Calculator")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b
```

FastMCPはこの単純なデコレータの背後で、複数の重要な処理を行っています：

1. **関数メタデータの抽出**：`inspect`モジュールを使用して関数のシグネチャ、戻り値の型、ドキュメント文字列を解析
2. **マネージャーへの登録**：`ToolManager`クラスに関数を`Tool`オブジェクトとして登録
3. **自動型変換**：Pydanticを利用した入力パラメータと戻り値の型変換・検証

実際の実装では、`server.py`の`tool`メソッドがデコレータを返し、そのデコレータが`ToolManager`に関数を登録します。このデコレータパターンは、関数に新しい機能を追加しながらも、オリジナルの関数は変更せず、デコレートされた関数の呼び出しセマンティクスを保持するという原則に従っています。

`mcp.server.fastmcp.tools.base`の`Tool.from_function`メソッドは、関数の詳細な情報を抽出し、`Tool`インスタンスを作成します。これには、関数が同期か非同期か、コンテキストパラメータが必要かどうかなどの情報も含まれます。

### 型ヒントからスキーマへの自動変換

FastMCPは、Pythonの型ヒントとdocstringを解析して、MCPプロトコルに必要なJSON Schemaを自動的に生成します。これにより、開発者は複雑なスキーマを手動で定義する必要がなくなります。

```python
from typing import List, Optional
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int
    email: Optional[str] = None

@mcp.tool()
def process_users(users: List[User]) -> str:
    """Process a list of users."""
    return f"Processed {len(users)} users"
```

このコードでは、FastMCPが自動的に以下を行います：

1. `User`クラスから適切なJSON Schemaを生成
2. `List[User]`を表すネストされたスキーマを生成
3. `process_users`の入力パラメータと戻り値のスキーマを生成

実装上は、`mcp.server.fastmcp.utilities.func_metadata`モジュールの`func_metadata`関数が中心的な役割を果たしています。この関数は、Pythonの型ヒントを解析し、Pydanticの`TypeAdapter`を使用してJSON Schemaを生成します。型変換の処理は以下のステップで行われます：

1. 関数のシグネチャを`inspect.signature`で解析
2. 各パラメータの型アノテーションを抽出
3. 型情報をPydanticの`TypeAdapter`でモデル化
4. モデルから`.json_schema()`メソッドでJSON Schemaを生成

複雑な型（ネストされた構造、ジェネリック型、ユニオン型など）も、Pydanticの型変換システムによって適切に処理されます。

### 非同期通信の管理

FastMCPは、asyncioを基盤とした非同期プログラミングモデルを採用し、同期処理と非同期処理を透過的に扱えるようにしています。リクエスト処理は方法に応じて適切なハンドラにディスパッチされます。

```python
# 同期関数も非同期関数も同じように定義できる
@mcp.tool()
def sync_tool(x: int) -> int:
    return x * 2

@mcp.tool()
async def async_tool(x: int) -> int:
    await asyncio.sleep(0.1)  # 非同期オペレーション
    return x * 3
```

FastMCPは関数が同期か非同期かを自動的に検出し、適切に実行します。この非同期処理の中核は、`mcp.server.fastmcp.tools.base`の`Tool.run`メソッドにあります：

```python
async def run(
    self,
    arguments: dict[str, Any],
    context: Context[ServerSessionT, LifespanContextT] | None = None,
) -> Any:
    """Run the tool with arguments."""
    try:
        return await self.fn_metadata.call_fn_with_arg_validation(
            self.fn,
            self.is_async,
            arguments,
            {self.context_kwarg: context}
            if self.context_kwarg is not None
            else None,
        )
    except Exception as e:
        raise ToolError(f"Error executing tool {self.name}: {e}") from e
```

この実装は以下の特徴を持っています：

1. 同期関数と非同期関数を統一的に扱う（`is_async`フラグを使用）
2. 引数の検証と型変換を行う
3. コンテキストオブジェクトを適切に注入する
4. エラー処理を統一的に行う

トランスポート間の抽象化は、`mcp.server.stdio`と`mcp.server.sse`モジュールによって提供されます。どちらも同じインターフェースを持つメモリオブジェクトストリームを提供し、低レベルの通信の詳細を隠蔽します。`FastMCP.run`メソッドは、指定されたトランスポートを自動的に初期化し、適切なストリームを確立します。

### コンテキスト提供

FastMCPは、各リクエスト処理に対して専用のコンテキストオブジェクトを提供し、ログ記録、進捗報告、リソースアクセスなどの機能を簡単に利用できるようにしています。これにより、ハンドラ関数は必要な機能にアクセスしやすくなっています。

```python
@mcp.tool()
def log_and_process(data: str, ctx: Context) -> str:
    # ログ記録
    ctx.info(f"Processing: {data}")
    ctx.debug("Debug information")
    
    # 進捗報告
    ctx.report_progress(50, 100)
    
    # リソースアクセス
    config = ctx.read_resource("config://settings")
    
    return f"Processed: {data}"
```

コンテキストオブジェクトの実装は、`mcp.server.fastmcp.server`モジュールの`Context`クラスで行われています。このクラスは、以下の機能を提供します：

1. **ロギング**：各種ログレベル（debug, info, warning, error）のメソッド
2. **進捗報告**：`report_progress`メソッドによる処理進捗の報告
3. **リソースアクセス**：`read_resource`メソッドによるリソースの読み取り
4. **リクエスト情報**：`client_id`, `request_id`などのプロパティ
5. **セッションアクセス**：`session`プロパティによるセッションへのアクセス

コンテキストの注入は型アノテーションを通じて行われます。`Tool.from_function`メソッドは、関数のパラメータに`Context`型のアノテーションがあるかどうかを検査し、あれば`context_kwarg`として記録します。実行時には、この情報を使って適切なコンテキストオブジェクトを注入します。

重要なのは、コンテキストが高度に型付けされていることです。`Context`クラスは、`ServerSessionT`と`LifespanContextT`のジェネリック型パラメータを持ち、IDE補完やタイプチェックを効果的にサポートします。

### URIベースのリソースシステム

FastMCPは、URIテンプレートを使った直感的なリソース定義と、効率的なパラメータ抽出機能を提供しています。リソースのURIパターンは正規表現に変換され、リクエスト時に効率的にマッチングされます。

```python
# 静的リソース
@mcp.resource("config://settings")
def get_settings() -> str:
    return "{ 'debug': true }"

# パラメータ化されたリソース
@mcp.resource("users://{user_id}/profile")
def get_user_profile(user_id: str) -> str:
    return f"Profile for user {user_id}"
```

URIベースのリソースシステムの中核は、`mcp.server.fastmcp.resources.templates`モジュールの`ResourceTemplate`クラスです。このクラスは、以下の機能を実装しています：

1. **URIテンプレート解析**：`uri_template`プロパティにテンプレート文字列を格納
2. **正規表現変換**：`matches`メソッドで、URIテンプレートを正規表現に変換し、マッチングを行う
3. **パラメータ抽出**：マッチした部分から名前付きグループでパラメータを抽出
4. **動的リソース生成**：`create_resource`メソッドで、抽出したパラメータを使って動的にリソースを生成

実装では、URIテンプレート内の`{param}`形式のパターンを、正規表現の名前付きキャプチャグループ`(?P<param>[^/]+)`に変換します。これにより、`/users/123/profile`のようなURIから`{"user_id": "123"}`のようなパラメータ辞書を抽出できます。

`ResourceManager`クラスは、登録されたテンプレートを管理し、URIリクエストがあるとすべてのテンプレートに対してマッチングを試みます。マッチしたテンプレートがあれば、そのテンプレートを使ってリソースを動的に生成します。

### lifespan管理の実装

FastMCPは、サーバーの起動時と終了時に共有リソースを効率的に管理するための、lifespanコンテキストマネージャをサポートしています。これにより、データベース接続などの共有リソースをツール間で効率的に管理できます。

```python
from contextlib import asynccontextmanager
from dataclasses import dataclass
from typing import AsyncIterator

from mcp.server.fastmcp import Context, FastMCP

@dataclass
class AppContext:
    db: Database

@asynccontextmanager
async def app_lifespan(server: FastMCP) -> AsyncIterator[AppContext]:
    # 起動時の処理
    db = await Database.connect()
    try:
        yield AppContext(db=db)
    finally:
        # 終了時の処理
        await db.disconnect()

# lifespanをサーバーに渡す
mcp = FastMCP("MyApp", lifespan=app_lifespan)

@mcp.tool()
def query_db(ctx: Context) -> str:
    # lifespanコンテキストにアクセス
    db = ctx.request_context.lifespan_context.db
    return db.query()
```

lifespanの実装は、`mcp.server.fastmcp.server`モジュールの`FastMCP`クラスの`__init__`メソッドと`lifespan_wrapper`関数によって行われています。主な実装ポイント：

1. **コンテキストマネージャ受け入れ**：`FastMCP`コンストラクタが`lifespan`パラメータを受け取る
2. **ラッパー関数**：低レベルサーバーに渡すため、`lifespan_wrapper`で適切にラップ
3. **コンテキスト保存**：lifespanの結果は`request_context.lifespan_context`として保存され、各リクエストで利用可能

lifespanコンテキストマネージャは非同期コンテキストマネージャ（`AbstractAsyncContextManager`）として実装され、`async with`構文でASGIアプリケーションのライフサイクルに合わせて実行されます。これにより、データベース接続、キャッシュ、外部APIクライアントなどの共有リソースが適切に初期化と終了処理を行えます。

エラー処理とグレースフルシャットダウンは、Pythonの`try`/`finally`ブロックによって保証されます。lifespanコンテキストマネージャの`finally`ブロックは、例外が発生した場合でも常に実行されるため、リソースのクリーンアップが保証されます。

## 処理フロー

FastMCPにおけるリクエスト処理の全体的なフローは、クライアントからのリクエストが到着してからレスポンスが返されるまでの一連のステップを含みます。

1. **クライアントリクエスト受信**
   - stdioまたはSSEトランスポートでJSON-RPCメッセージを受信
   - `JSONRPCMessage`オブジェクトとして解析

2. **ディスパッチ処理**
   - `Server._handle_message`が受信メッセージを処理
   - リクエストかノーティフィケーションかを判断
   - リクエストコンテキストを作成・設定（`request_ctx`コンテキスト変数）

3. **ハンドラー呼び出し**
   - メッセージタイプに応じたハンドラーを検索
   - リクエストパラメータを解析・検証
   - ハンドラー関数を呼び出し

4. **FastMCP層での処理**
   - `ToolManager`/`ResourceManager`/`PromptManager`が機能処理を担当
   - 型変換と引数検証
   - コンテキスト注入（必要に応じて）
   - 同期/非同期関数の適切な実行

5. **結果処理**
   - 関数の戻り値をMCPプロトコル形式に変換
   - JSONレスポンスを生成

6. **クライアントへの応答**
   - JSON-RPCレスポンスをシリアライズ
   - トランスポートを通じてクライアントに送信

以下は、ツール呼び出しの簡略化されたシーケンス図です：

```
クライアント         FastMCP                  ToolManager              Tool
    |                   |                         |                     |
    |---call_tool------>|                         |                     |
    |                   |---call_tool------------>|                     |
    |                   |                         |---get_tool--------->|
    |                   |                         |<--Tool instance-----|
    |                   |                         |---run-------------->|
    |                   |                         |                     |---validate args-->|
    |                   |                         |                     |---execute fn----->|
    |                   |                         |                     |<--result----------|
    |                   |                         |<--result------------|
    |                   |<--result----------------|
    |<--result----------|                         |                     |
```

FastMCPの抽象化レイヤーは、低レベルのプロトコル詳細（メッセージフォーマットやエンコーディング）、型変換と検証、エラー処理、非同期通信などを処理し、開発者が本質的な機能実装に集中できるようにします。

## まとめ

FastMCPは、MCPプロトコルの複雑さを様々なレベルで抽象化し、開発者が本質的な機能実装に集中できるようにします。デコレータ、型ヒント解析、非同期処理、コンテキスト提供などの技術を巧みに組み合わせることで、簡潔で直感的なAPIと強力な内部実装を両立させています。

これらの抽象化により、通常は多くのコードが必要な複雑なMCPサーバーが、少ないコードで実装可能になります。また、独自のMCPサーバーを実装する場合は、FastMCPの抽象化アプローチを参考にすることで、効率的な設計が可能になるでしょう。

