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
### MCPの通信プロトコル
MCP の通信は、JSON-RPC 2.0 プロトコルに基づいています。これはシンプルなテキストベースのプロトコルで、クライアントとサーバー間での関数呼び出しを可能にします。MCP通信は標準入出力（stdio）または HTTP with Server-Sent Events（SSE）を介して行われ、全てのメッセージはJSON形式で送受信されます。例えば、ツール呼び出しのリクエストは `{"jsonrpc": "2.0", "method": "call_tool", "params": {"name": "add", "arguments": {"a": 5, "b": 3}}, "id": 123}` のような形式となります。詳細は[JSON-RPC 2.0仕様](https://www.jsonrpc.org/specification)を参照してください。

### 通信トランスポート
MCPは２種類のトランスポートをサポートしています。stdioトランスポートはもっともシンプルで、標準入出力を使って1行ごとにJSON-RPCメッセージを送受信します。これはコマンドラインツールやサブプロセスとの統合に適しています。一方、SSE（Server-Sent Events）トランスポートはHTTPベースで、サーバーからクライアントへの一方向のリアルタイム通信が可能です。クライアントはGETリクエストでSSE接続を確立し、サーバーはこの接続を通じてメッセージを送信します。クライアントからサーバーへのメッセージは別のPOSTリクエストで行われます。SSEについては[MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)に詳しい説明があります。

### ASGI対応
MCPサーバーはASGI（Asynchronous Server Gateway Interface）に対応しており、FastAPI、Starlette、Uvicornなど主要なPythonウェブフレームワークと簡単に統合できます。ASGIは非同期Webアプリケーションのための標準インターフェースで、`async def app(scope, receive, send)`という形式で実装されます。FastMCPはStarletteを使用してASGIアプリケーションを構築し、SSEエンドポイントとHTTPメッセージングを提供します。ASGIの詳細は[ASGI仕様](https://asgi.readthedocs.io/en/latest/)をご覧ください。

## MCPの処理フロー
一般的なMCPサーバーの処理の例として、SSE接続における**初回接続確立からツールの使用**までのMCPサーバーとクライアント間のデータフローを解説します。

### 初回接続確立時のフロー
1. クライアント接続開始：
 - クライアントがSSEエンドポイント(`/sse`)に接続
 - サーバーはSSE接続を確立してセッションIDを生成
2. 初期化リクエスト：
 - クライアントが`initialize`リクエストを送信
 - 内容：`InitializeRequestParams`（プロトコルバージョン、クライアント情報、サポート機能）
 ``` json
    {
     "jsonrpc": "2.0",
     "id": "req-1",
     "method": "initialize",
     "params": {
       "protocolVersion": "2024-11-05",
       "capabilities": { ... },
       "clientInfo": {
         "name": "クライアント名",
         "version": "1.0.0"
       }
     }
   }
 ```

3. 初期化レスポンス：
 - サーバーが`InitializeResult`を返信
 - 内容：プロトコルバージョン、サーバー情報、サポート機能
 ``` json
    {
     "jsonrpc": "2.0",
     "id": "req-1",
     "result": {
       "protocolVersion": "2024-11-05",
       "capabilities": {
         "tools": { "listChanged": true },
         "resources": { "subscribe": false, "listChanged": true },
         "prompts": { "listChanged": true }
       },
       "serverInfo": {
         "name": "FastMCP",
         "version": "1.0.0"
       },
       "instructions": "オプションの説明テキスト"
     }
   }
 ```
4. 初期化完了通知：
 - クライアントが`initialized`通知を送信
 ``` json
    {
     "jsonrpc": "2.0",
     "method": "notifications/initialized",
     "params": {}
   }
 ```
### ツール呼び出しまでのフロー
1. ツール一覧取得：
 - クライアントが`tools/list`リクエストを送信
 ``` json
    {
     "jsonrpc": "2.0",
     "id": "req-2",
     "method": "tools/list",
     "params": {}
   }
 ```
2. ツール一覧レスポンス：
 - サーバーが`ListToolsResult`を返信
 - 各ツールの名前、説明、入力スキーマを含む
 ``` json
    {
     "jsonrpc": "2.0",
     "id": "req-2",
     "result": {
       "tools": [
         {
           "name": "get_weather",
           "description": "都市の天気情報を取得する",
           "inputSchema": {
             "type": "object",
             "properties": {
               "city": { "type": "string" }
             },
             "required": ["city"]
           }
         }
       ]
     }
   }
 ```
3. ツール呼び出し：
 - クライアントが`tools/call`リクエストを送信
 - 呼び出すツール名と引数を指定
 ``` json
    {
     "jsonrpc": "2.0",
     "id": "req-3",
     "method": "tools/call",
     "params": {
       "name": "get_weather",
       "arguments": {
         "city": "東京"
       },
       "_meta": {
         "progressToken": "progress-1"
       }
     }
   }
 ```
4. 進捗通知（オプション）：
 - サーバーが`notifications/progress`通知を送信
 ``` json
    {
     "jsonrpc": "2.0",
     "method": "notifications/progress",
     "params": {
       "progressToken": "progress-1",
       "progress": 50,
       "total": 100
     }
   }
 ```
5. ツール呼び出し結果：
 - サーバーが`CallToolResult`を返信
 - テキスト、画像、埋め込みリソースなどの形式で結果を返す
 ``` json
    {
     "jsonrpc": "2.0",
     "id": "req-3",
     "result": {
       "content": [
         {
           "type": "text",
           "text": "東京の天気: 晴れ、気温: 25°C"
         }
       ],
       "isError": false
     }
   }
 ```

### 一般的なPythonモジュールによる実装
FastMCPやpython-SDKを使わずに、標準ライブラリと一般的なPythonパッケージだけで前述の処理フローを実装した最小限のMCPサーバー例を示します：

```python
import asyncio
import json
import sys
import uuid
from typing import Any, Dict, List, Optional, Union

# JSON-RPC 2.0のメッセージ構造を定義
class JsonRpcRequest:
    def __init__(self, method: str, params: Any, id: Union[str, int]):
        self.jsonrpc = "2.0"
        self.method = method
        self.params = params
        self.id = id
    
    @classmethod
    def from_dict(cls, data: Dict[str, Any]):
        return cls(data.get("method"), data.get("params"), data.get("id"))
    
    def to_dict(self):
        return {
            "jsonrpc": self.jsonrpc,
            "method": self.method,
            "params": self.params,
            "id": self.id
        }

class JsonRpcResponse:
    def __init__(self, result: Any, id: Union[str, int]):
        self.jsonrpc = "2.0"
        self.result = result
        self.id = id
    
    def to_dict(self):
        return {
            "jsonrpc": self.jsonrpc,
            "result": self.result,
            "id": self.id
        }

class JsonRpcNotification:
    def __init__(self, method: str, params: Any):
        self.jsonrpc = "2.0"
        self.method = method
        self.params = params
    
    def to_dict(self):
        return {
            "jsonrpc": self.jsonrpc,
            "method": self.method,
            "params": self.params
        }

class JsonRpcError:
    def __init__(self, error: Dict[str, Any], id: Union[str, int]):
        self.jsonrpc = "2.0"
        self.error = error
        self.id = id
    
    def to_dict(self):
        return {
            "jsonrpc": self.jsonrpc,
            "error": self.error,
            "id": self.id
        }

# MCPサーバー実装
class MinimalMcpServer:
    def __init__(self, name: str, version: str = "1.0.0", instructions: str = None):
        self.name = name
        self.version = version
        self.instructions = instructions
        self.tools = {}
        self.resources = {}
        self.sessions = {}
        self.protocol_version = "2024-11-05"
        
        # メソッドハンドラーの登録
        self.method_handlers = {
            "initialize": self.handle_initialize,
            "tools/list": self.handle_list_tools,
            "tools/call": self.handle_call_tool,
            "resources/list": self.handle_list_resources,
            "resources/read": self.handle_read_resource,
        }
    
    # 初期化ハンドラー
    async def handle_initialize(self, params: Dict[str, Any], request_id: str) -> Dict[str, Any]:
        client_info = params.get("clientInfo", {})
        client_capabilities = params.get("capabilities", {})
        
        # セッション情報を保存
        session_id = str(uuid.uuid4())
        self.sessions[session_id] = {
            "client_info": client_info,
            "capabilities": client_capabilities
        }
        
        # サーバー情報と機能を返す
        return {
            "protocolVersion": self.protocol_version,
            "serverInfo": {
                "name": self.name,
                "version": self.version
            },
            "capabilities": {
                "tools": {"listChanged": True},
                "resources": {"subscribe": False, "listChanged": True},
                "prompts": {"listChanged": True}
            },
            "instructions": self.instructions
        }
    
    # ツール登録
    def register_tool(self, name: str, description: str, input_schema: Dict[str, Any], handler_func):
        self.tools[name] = {
            "name": name, 
            "description": description,
            "inputSchema": input_schema,
            "handler": handler_func
        }
    
    # ツール一覧ハンドラー
    async def handle_list_tools(self, params: Dict[str, Any], request_id: str) -> Dict[str, Any]:
        tools = []
        for name, tool in self.tools.items():
            tools.append({
                "name": tool["name"],
                "description": tool["description"],
                "inputSchema": tool["inputSchema"]
            })
        return {"tools": tools}
    
    # ツール呼び出しハンドラー
    async def handle_call_tool(self, params: Dict[str, Any], request_id: str) -> Dict[str, Any]:
        tool_name = params.get("name")
        arguments = params.get("arguments", {})
        meta = params.get("_meta", {})
        progress_token = meta.get("progressToken")
        
        if tool_name not in self.tools:
            raise ValueError(f"Unknown tool: {tool_name}")
        
        tool = self.tools[tool_name]
        
        # 進捗通知を送信（オプション）
        if progress_token:
            await self.send_progress_notification(progress_token, 50, 100)
        
        # ツール実行
        result = await tool["handler"](arguments)
        
        # 結果をフォーマット
        if isinstance(result, str):
            content = [{"type": "text", "text": result}]
        elif isinstance(result, dict) and "type" in result:
            content = [result]
        elif isinstance(result, list):
            content = result
        else:
            content = [{"type": "text", "text": str(result)}]
        
        return {
            "content": content,
            "isError": False
        }
    
    # 進捗通知の送信
    async def send_progress_notification(self, token: str, progress: int, total: int = None):
        notification = JsonRpcNotification(
            "notifications/progress",
            {
                "progressToken": token,
                "progress": progress,
                "total": total
            }
        )
        await self.send_message(notification.to_dict())
    
    # リソース関連のハンドラーは省略...
    
    # リクエスト処理のメインループ
    async def process_request(self, request_data: Dict[str, Any]):
        try:
            method = request_data.get("method")
            params = request_data.get("params", {})
            request_id = request_data.get("id")
            
            # メソッドの種類に基づいて処理
            if method == "notifications/initialized":
                # 初期化完了通知の処理
                return None
            
            if method not in self.method_handlers:
                error = {"code": -32601, "message": f"Method not found: {method}"}
                return JsonRpcError(error, request_id).to_dict()
            
            # ハンドラー呼び出し
            handler = self.method_handlers[method]
            result = await handler(params, request_id)
            
            # レスポンス返却
            return JsonRpcResponse(result, request_id).to_dict()
            
        except Exception as e:
            error = {"code": -32603, "message": f"Internal error: {str(e)}"}
            return JsonRpcError(error, request_data.get("id")).to_dict()
    
    # メッセージ送信
    async def send_message(self, message: Dict[str, Any]):
        print(json.dumps(message), flush=True)
    
    # stdin/stdoutによるMCPサーバー実行
    async def run_stdio(self):
        try:
            while True:
                # 入力を読み取り
                line = await asyncio.get_event_loop().run_in_executor(None, sys.stdin.readline)
                if not line:
                    break
                
                # JSONパース
                try:
                    request_data = json.loads(line)
                    
                    # リクエスト処理
                    response = await self.process_request(request_data)
                    if response:
                        await self.send_message(response)
                        
                except json.JSONDecodeError:
                    error = {"code": -32700, "message": "Parse error"}
                    await self.send_message(JsonRpcError(error, None).to_dict())
                    
        except Exception as e:
            print(f"Server error: {e}", file=sys.stderr)

# 使用例
async def main():
    # サーバー作成
    server = MinimalMcpServer(
        name="MinimalMCP", 
        version="0.1.0",
        instructions="Minimal MCP server implementation example"
    )
    
    # 天気情報ツールの登録
    server.register_tool(
        name="get_weather",
        description="都市の天気情報を取得する",
        input_schema={
            "type": "object",
            "properties": {
                "city": {"type": "string"}
            },
            "required": ["city"]
        },
        handler_func=weather_handler
    )
    
    # ツールハンドラー実装
    async def weather_handler(arguments):
        city = arguments.get("city", "不明")
        # 実際には天気APIを呼び出すなどの処理
        return f"{city}の天気: 晴れ、気温: 25°C"
    
    # サーバー実行
    await server.run_stdio()

if __name__ == "__main__":
    asyncio.run(main())
```

この実装では、「MCPの処理フロー」セクションで説明した初期化から始まるツール呼び出しのフローに対応しています。具体的には:

1. `initialize`メソッドでクライアント接続を受け付け、サーバー情報とサポート機能を返す
2. `tools/list`と`tools/call`を処理する専用ハンドラーを用意
3. 進捗通知のサポート
4. JSON-RPC 2.0に準拠したリクエスト/レスポンス処理

ただし、この実装は最小限のものであり、エラー処理、型検証、セッション管理、リソース管理などの多くの詳細は省略または簡略化されています。FastMCPは、これらすべての複雑な処理を抽象化し、開発者が核となる機能実装に集中できるよう支援します。

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

