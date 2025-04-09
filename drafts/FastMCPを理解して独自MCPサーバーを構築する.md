# FastMCPを理解して独自MCPサーバーを構築する

## 概要
この記事では、Model Context Protocol (MCP)の高レベル抽象化ライブラリであるFastMCPの内部実装を解析し、その知見を基にPythonの標準ライブラリ(特にasyncio)を活用して独自のMCPサーバーを構築する方法を解説します。FastMCPの便利さを理解しながらも、その抽象化レイヤーを使わずに低レベルでMCPサーバーを実装することで、プロトコルへの理解を深め、より柔軟なカスタマイズが可能になります。

## 詳細

### MCPプロトコルとは

Model Context Protocol (MCP)は、大規模言語モデル(LLM)アプリケーションと外部ツールやデータソース間のインタラクションを標準化するためのオープンプロトコルです。従来、各AIアプリケーションと外部ツールの連携は個別に実装する必要がありましたが、MCPは標準的なインターフェースを提供することで、この問題を解決します。

MCPの基本的なアーキテクチャは以下の3つのコンポーネントで構成されています：

1. **ホスト(Host)**: ユーザーが直接対話するLLMアプリケーション（Claude Desktop、CursorなどのIDE、カスタムエージェントなど）
2. **クライアント(Client)**: ホスト内で動作し、特定のMCPサーバーとの接続を管理するコンポーネント
3. **サーバー(Server)**: 外部ツール、リソース、プロンプトを提供するプログラム

この構成により、M個のAIアプリケーションとN個の外部システムの連携において、M×NではなくM+N個の実装で済むようになります。

### FastMCPとは

FastMCPは`mcp-python-sdk`に含まれる高レベル抽象化ライブラリで、MCPサーバー開発を簡略化します。元々は独立したプロジェクト（jlowin/fastmcp）として開発されていましたが、その有用性が認められて公式SDKに統合されました。FastMCPの主な目的は「高速でPythonic」なインターフェースを提供することにより、MCPサーバーの開発プロセスを大幅に簡略化することです。

基本的な使用例：

```python
from mcp import FastMCP

mcp = FastMCP("My Server Name")

@mcp.tool
async def add(a: int, b: int) -> int:
    """2つの数値を足します"""
    return a + b

if __name__ == "__main__":
    mcp.run(transport='stdio')
```

### FastMCPの内部実装を理解する

FastMCPが内部で行っている主な処理を理解することで、独自実装の設計指針が明確になります。

#### 1. コアコンポーネント

FastMCPの主要コンポーネントは次の3つです：

**FastMCP クラス**:
- サーバーインスタンスを表す中心的なクラス
- デコレータの基点となり、関数をサーバー機能として登録
- 接続受付、メッセージルーティング、応答送信などを管理
- 並行リクエスト処理やキャンセル処理などの高度な機能も提供

**デコレータ (@mcp.tool, @mcp.resource, @mcp.prompt)**:
- Pythonの関数をMCPプリミティブとして公開するための抽象化メカニズム
- 関数のシグネチャと型ヒントを解析し、MCPスキーマを自動生成
- リクエスト処理時の引数検証や結果シリアライズを行うラッパー関数を生成
- 関数が非同期（async def）かどうかを自動検出して適切に実行

**Context オブジェクト**:
- ハンドラ関数が実行時にアクセスできるセッション情報と機能
- 進捗報告(report_progress)やログ出力(debug, info)などの機能を提供
- 他のリソースへのアクセス(read_resource)などを可能にする
- リクエスト固有の状態や共有リソース（データベース接続など）へのアクセスを抽象化

#### 2. リクエスト処理フロー

FastMCPがリクエストを処理する内部フローは以下のように推測されます：

1. トランスポート層でデータ受信（stdin/HTTPなど）
2. メッセージフレーミング処理（例：改行区切り）
3. JSONデシリアライズと基本的なJSON-RPC検証
4. メソッド名とパラメータの抽出
5. 適切なハンドラ関数へのディスパッチ
6. 引数の検証と変換（Pydanticなどを利用）
7. Contextオブジェクトの生成（必要な場合）
8. ハンドラ関数の実行（非同期の場合はasyncioで管理）
9. 結果の取得
10. 結果のシリアライズ（エラーの場合は適切な処理）
11. JSON-RPCレスポンスの構築
12. トランスポート層での応答送信

#### 3. 抽象化されている機能

FastMCPが抽象化している主な機能は以下の通りです：

- MCP接続ライフサイクル管理（初期化ハンドシェイク、能力ネゴシエーション）
- JSON-RPCメッセージ解析/シリアライズ
- メッセージフレーミング処理
- メソッドディスパッチ機構
- パラメータ検証/シリアライズ
- MCPスキーマ自動生成
- Contextオブジェクト提供
- エラーハンドリング
- 非同期実行管理
- トランスポート層管理

#### 4. FastMCPの抽象化技術と内部メカニズム

FastMCPが使用している主要な抽象化技術を詳しく見ていきます：

##### a. デコレータによる宣言的定義

FastMCPは、デコレータ (@mcp.tool, @mcp.resource, @mcp.prompt) を使用して、Pythonの関数をMCPプリミティブとして登録します。これにより開発者は、クラスを明示的に継承したり登録関数を呼び出したりする代わりに、関数に単純にデコレータを付与するだけでサーバーに機能を追加できる宣言的なプログラミングスタイルが可能になります。

```python
@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b
```

内部的には、このデコレータは `Tool` クラスを使用して関数を登録します。`Tool` クラスは、元の関数と関連するメタデータ（名前、説明、スキーマなど）をカプセル化し、実行時には適切な検証と変換を行います。

##### b. 型ヒントとPydanticによる自動スキーマ生成

FastMCPは、Pythonの型ヒントを活用して、ツールのパラメータ用のJSONスキーマを自動的に生成します。これにより、開発者はスキーマを手動で定義する必要がなくなります。

内部的には、`inspect` モジュールを使用して関数の引数情報と型ヒントを抽出し、Pydanticモデルを動的に生成します。このモデルは、引数の検証と型変換に使用されるだけでなく、MCP仕様で必要なJSONスキーマ形式への変換にも利用されます。

例えば、`a: int, b: int` という型ヒントから、以下のようなJSONスキーマが自動生成されます：

```json
{
  "type": "object",
  "properties": {
    "a": {"type": "integer"},
    "b": {"type": "integer"}
  },
  "required": ["a", "b"]
}
```

##### c. 非同期プログラミングの抽象化

FastMCPは、Pythonの `asyncio` を全面的に活用し、ツール、リソース、ライフサイクル管理など様々なコンポーネントで非同期処理をサポートしています。開発者は `async def` で関数を定義するだけで、FastMCPが適切に非同期実行します。

内部的には、`inspect.iscoroutinefunction()` を使用して関数が非同期かどうかを検出し、非同期関数の場合は適切に `await` して実行します。これにより、I/O処理を含む長時間実行される関数でもサーバーがブロックされることなく、効率的な並行処理が可能になります。

##### d. コンテキスト注入によるアクセス制御

FastMCPは、`Context` オブジェクトを経由してサーバー機能へのアクセスを提供します。ハンドラ関数に `ctx: Context` パラメータを追加するだけで、実行時にそのパラメータに `Context` オブジェクトが自動的に注入されます。

```python
@mcp.tool()
def get_data(query: str, ctx: Context) -> str:
    ctx.info(f"Processing query: {query}")  # ログ出力
    return f"Results for {query}"
```

内部的には、関数シグネチャを解析して `Context` 型のパラメータを検出し、実行時に適切な `Context` インスタンスを注入します。これにより、ハンドラ関数とサーバー内部の詳細実装との結合が疎になり、テスト容易性とコードのモジュール性が向上します。

##### e. ライフサイクル管理と状態共有

FastMCPは、`lifespan` 引数を通じてサーバーのライフサイクルに連動したリソース管理（データベース接続など）をサポートします。

```python
@asynccontextmanager
async def app_lifespan(server: FastMCP) -> AsyncIterator[AppContext]:
    db = Database()  # データベース初期化
    try:
        await db.connect()
        yield AppContext(db=db)
    finally:
        await db.disconnect()

mcp = FastMCP("My App", lifespan=app_lifespan)
```

内部的には、サーバーの起動時と停止時に適切なタイミングでこのコンテキストマネージャを呼び出し、yield されたコンテキストオブジェクトへのアクセスを `ctx.request_context.lifespan_context` 経由で提供します。これにより、リソースのライフサイクル管理が簡素化され、状態共有が安全に行えます。

##### f. プロトコルとトランスポートの抽象化

FastMCPは、低レベルなMCPプロトコルの詳細（JSON-RPCメッセージングやトランスポート層）を完全に抽象化します。開発者は、`mcp.run(transport='stdio')` のように、使用するトランスポートを指定するだけです。

内部的には、FastMCPがJSON-RPCメッセージの解析・構築、トランスポート層のセットアップと通信処理、メソッドディスパッチ、エラーハンドリングなどをすべて処理します。これにより、開発者はプロトコルの詳細を理解しなくても、MCPサーバーを迅速に構築できます。

### 独自MCPサーバーの実装

以下に、FastMCPを使わずにPythonの標準ライブラリ（特にasyncio）を用いてMCPサーバーを構築する方法を示します。

#### 1. 基本構造

```python
import asyncio
import json
import sys
from typing import Dict, Any, Callable, Optional, Union

# サーバークラス
class MCPServer:
    def __init__(self, name: str):
        self.name = name
        self.tools: Dict[str, Callable] = {}
        self.resources: Dict[str, Callable] = {}
        self.prompts: Dict[str, Callable] = {}
        self.server_capabilities = {
            "tools": {"listChanged": False},
            "resources": {"listChanged": False},
            "prompts": {"get": True if self.prompts else False}
        }
        
    # ツール登録メソッド
    def register_tool(self, name: str, func: Callable, description: str, schema: Dict):
        self.tools[name] = {
            "func": func,
            "description": description,
            "schema": schema
        }
    
    # リソース登録メソッド
    def register_resource(self, uri: str, func: Callable):
        self.resources[uri] = func
    
    # サーバー起動メソッド（stdioトランスポート）
    async def run_stdio(self):
        # 標準入出力の設定
        reader = asyncio.StreamReader()
        protocol = asyncio.StreamReaderProtocol(reader)
        await asyncio.get_running_loop().connect_read_pipe(
            lambda: protocol, sys.stdin)
        
        writer_transport, writer_protocol = await asyncio.get_running_loop().connect_write_pipe(
            asyncio.streams.FlowControlMixin, sys.stdout)
        writer = asyncio.StreamWriter(
            writer_transport, writer_protocol, None, asyncio.get_running_loop())
        
        print("MCP Server starting (stdio)...", file=sys.stderr)
        
        # メインループ
        try:
            await self._main_loop(reader, writer)
        except Exception as e:
            print(f"Server error: {e}", file=sys.stderr)
        finally:
            writer.close()
    
    # メインループ処理
    async def _main_loop(self, reader, writer):
        # 初期化待機
        initialized = False
        
        while True:
            # メッセージ読み取り
            line = await reader.readline()
            if not line:
                break  # 接続終了
                
            try:
                # JSONパース
                request = json.loads(line.decode('utf-8'))
                
                # JSON-RPC検証
                if "jsonrpc" not in request or request["jsonrpc"] != "2.0":
                    await self._send_error(writer, 
                        request.get("id"), -32600, "Invalid Request: Not a valid JSON-RPC 2.0 message")
                    continue
                
                # メソッド処理
                if "method" in request:
                    method = request["method"]
                    params = request.get("params", {})
                    
                    # 初期化ハンドシェイク
                    if method == "initialize":
                        if not initialized:
                            response = {
                                "jsonrpc": "2.0",
                                "id": request["id"],
                                "result": {
                                    "protocolVersion": "2024-11-05",
                                    "capabilities": self.server_capabilities
                                }
                            }
                            await self._send_response(writer, response)
                        else:
                            await self._send_error(writer, 
                                request.get("id"), -32600, "Server already initialized")
                                
                    elif method == "initialized":
                        # 通知なので応答不要
                        initialized = True
                        print("Client initialized connection", file=sys.stderr)
                    
                    # 初期化済みの場合のみ他のメソッドを処理
                    elif initialized:
                        if method == "tools/call":
                            await self._handle_tool_call(writer, request)
                        elif method == "resources/read":
                            await self._handle_resource_read(writer, request)
                        elif method == "tools/list":
                            await self._handle_tools_list(writer, request)
                        elif method == "resources/list":
                            await self._handle_resources_list(writer, request)
                        elif method == "shutdown":
                            # シャットダウン処理
                            response = {
                                "jsonrpc": "2.0",
                                "id": request["id"],
                                "result": None
                            }
                            await self._send_response(writer, response)
                            break
                        else:
                            # 未知のメソッド
                            await self._send_error(writer, 
                                request.get("id"), -32601, f"Method not found: {method}")
                    else:
                        # 初期化前の他メソッド呼び出し
                        await self._send_error(writer, 
                            request.get("id"), -32002, "Server not initialized")
                        
            except json.JSONDecodeError:
                # JSONパースエラー
                await self._send_error(writer, None, -32700, "Parse error")
            except Exception as e:
                # その他の内部エラー
                request_id = request.get("id") if 'request' in locals() else None
                await self._send_error(writer, request_id, -32603, f"Internal error: {str(e)}")
    
    # ツール呼び出しハンドラ
    async def _handle_tool_call(self, writer, request):
        request_id = request.get("id")
        params = request.get("params", {})
        tool_name = params.get("name")
        arguments = params.get("arguments", {})
        
        if tool_name not in self.tools:
            await self._send_error(writer, request_id, -32601, f"Tool not found: {tool_name}")
            return
            
        tool = self.tools[tool_name]
        try:
            # ツール関数実行
            result = await tool["func"](**arguments) if asyncio.iscoroutinefunction(tool["func"]) else tool["func"](**arguments)
            
            # 結果をレスポンスとして返す
            response = {
                "jsonrpc": "2.0",
                "id": request_id,
                "result": {
                    "content": result
                }
            }
            await self._send_response(writer, response)
        except Exception as e:
            # ツール実行エラー
            response = {
                "jsonrpc": "2.0",
                "id": request_id,
                "result": {
                    "isError": True,
                    "content": str(e)
                }
            }
            await self._send_response(writer, response)
    
    # レスポンス送信ヘルパー
    async def _send_response(self, writer, response):
        response_json = json.dumps(response)
        writer.write(response_json.encode('utf-8') + b'\n')
        await writer.drain()
    
    # エラー送信ヘルパー
    async def _send_error(self, writer, request_id, code, message):
        error_response = {
            "jsonrpc": "2.0",
            "id": request_id,
            "error": {
                "code": code,
                "message": message
            }
        }
        await self._send_response(writer, error_response)
```

#### 2. 実装のポイント

##### JSON-RPC 2.0メッセージング

MCP通信の基本はJSON-RPC 2.0です。リクエスト、レスポンス、通知の3種類のメッセージタイプがあります。

- **リクエスト**: `{"jsonrpc": "2.0", "id": "request-id", "method": "method-name", "params": {...}}`
- **レスポンス**: `{"jsonrpc": "2.0", "id": "request-id", "result": {...}}` または `{"jsonrpc": "2.0", "id": "request-id", "error": {...}}`
- **通知**: `{"jsonrpc": "2.0", "method": "method-name", "params": {...}}` (idなし)

##### MCPライフサイクル

MCPサーバーの接続ライフサイクルは、初期化、運用、シャットダウンの3段階です。

1. **初期化**: クライアントからの`initialize`リクエスト受信→サーバーの能力を返す→クライアントからの`initialized`通知
2. **運用**: ツール呼び出し、リソース読み取りなどのメソッド処理
3. **シャットダウン**: `shutdown`リクエスト処理→接続終了

##### トランスポートメカニズム

MCPは主に2つのトランスポートをサポートしています：

- **stdio**: 標準入出力を使った通信。メッセージは改行(`\n`)で区切られます。
- **HTTP/SSE**: HTTPリクエストとServer-Sent Eventsを使った通信。

##### エラーハンドリング

JSON-RPCには標準的なエラーコードが定義されています：

- `-32700`: Parse error (JSONパースエラー)
- `-32600`: Invalid Request (無効なリクエスト)
- `-32601`: Method not found (存在しないメソッド)
- `-32602`: Invalid params (無効なパラメータ)
- `-32603`: Internal error (内部エラー)

### 実装例：シンプルなMCPサーバー

以下に、minimal-mcpと呼ぶシンプルなMCPサーバーの実装例を示します。

```python
# minimal_mcp.py
import asyncio
import json
import sys
from typing import Dict, Any, Optional, Callable, Awaitable, Union

class MinimalMCP:
    def __init__(self, name: str):
        self.name = name
        self.tools = {}
        
    def tool(self, func):
        """ツール登録用デコレータ"""
        self.tools[func.__name__] = {
            "func": func,
            "description": func.__doc__ or "",
            "schema": self._extract_schema(func)
        }
        return func
    
    def _extract_schema(self, func) -> Dict[str, Any]:
        """関数から簡易的なJSONスキーマを生成"""
        import inspect
        sig = inspect.signature(func)
        properties = {}
        required = []
        
        for name, param in sig.parameters.items():
            if name == 'ctx':  # Contextパラメータはスキップ
                continue
                
            param_type = param.annotation
            type_name = "string"  # デフォルト
            
            if param_type is int:
                type_name = "integer"
            elif param_type is float:
                type_name = "number"
            elif param_type is bool:
                type_name = "boolean"
            
            properties[name] = {"type": type_name}
            
            if param.default is inspect.Parameter.empty:
                required.append(name)
        
        return {
            "type": "object",
            "properties": properties,
            "required": required
        }
    
    async def run(self):
        """サーバーを起動 (stdio)"""
        # stdioの設定
        reader = asyncio.StreamReader()
        protocol = asyncio.StreamReaderProtocol(reader)
        await asyncio.get_running_loop().connect_read_pipe(
            lambda: protocol, sys.stdin)
        
        writer_transport, writer_protocol = await asyncio.get_running_loop().connect_write_pipe(
            asyncio.streams.FlowControlMixin, sys.stdout)
        writer = asyncio.StreamWriter(
            writer_transport, writer_protocol, None, asyncio.get_running_loop())
        
        print(f"MinimalMCP Server '{self.name}' starting...", file=sys.stderr)
        
        # メインループ
        initialized = False
        
        while True:
            line = await reader.readline()
            if not line:
                break
                
            try:
                # JSONパース
                request = json.loads(line.decode('utf-8'))
                
                # 基本検証
                if "jsonrpc" not in request or request["jsonrpc"] != "2.0":
                    await self._send_error(writer, 
                        request.get("id"), -32600, "Invalid Request")
                    continue
                
                # メソッド処理
                if "method" in request:
                    request_id = request.get("id")
                    method = request["method"]
                    params = request.get("params", {})
                    
                    # 初期化処理
                    if method == "initialize":
                        if not initialized:
                            # 能力情報を返す
                            response = {
                                "jsonrpc": "2.0",
                                "id": request_id,
                                "result": {
                                    "protocolVersion": "2024-11-05",
                                    "capabilities": {
                                        "tools": {"listChanged": False}
                                    }
                                }
                            }
                            await self._send_response(writer, response)
                        else:
                            await self._send_error(writer, 
                                request_id, -32600, "Server already initialized")
                                
                    elif method == "initialized":
                        # 通知なので応答不要
                        initialized = True
                        print("Client initialized", file=sys.stderr)
                    
                    # 初期化済みの場合のみ他のメソッドを処理
                    elif initialized:
                        if method == "tools/list":
                            # ツール一覧を返す
                            tools_list = []
                            for name, tool_info in self.tools.items():
                                tools_list.append({
                                    "name": name,
                                    "description": tool_info["description"],
                                    "inputSchema": tool_info["schema"]
                                })
                            
                            response = {
                                "jsonrpc": "2.0",
                                "id": request_id,
                                "result": {
                                    "tools": tools_list
                                }
                            }
                            await self._send_response(writer, response)
                            
                        elif method == "tools/call":
                            # ツール呼び出し
                            tool_name = params.get("name")
                            arguments = params.get("arguments", {})
                            
                            if tool_name not in self.tools:
                                await self._send_error(writer, 
                                    request_id, -32601, f"Tool not found: {tool_name}")
                                continue
                                
                            try:
                                # ツール関数実行
                                tool_func = self.tools[tool_name]["func"]
                                result = await tool_func(**arguments) if asyncio.iscoroutinefunction(tool_func) else tool_func(**arguments)
                                
                                # 結果を返す
                                response = {
                                    "jsonrpc": "2.0",
                                    "id": request_id,
                                    "result": {
                                        "content": result
                                    }
                                }
                                await self._send_response(writer, response)
                            except Exception as e:
                                # ツール実行エラー
                                response = {
                                    "jsonrpc": "2.0",
                                    "id": request_id,
                                    "result": {
                                        "isError": True,
                                        "content": str(e)
                                    }
                                }
                                await self._send_response(writer, response)
                        
                        elif method == "shutdown":
                            # シャットダウン
                            response = {
                                "jsonrpc": "2.0",
                                "id": request_id,
                                "result": None
                            }
                            await self._send_response(writer, response)
                            break
                            
                        else:
                            # 未知のメソッド
                            await self._send_error(writer, 
                                request_id, -32601, f"Method not found: {method}")
                    else:
                        # 初期化前の他メソッド呼び出し
                        await self._send_error(writer, 
                            request_id, -32002, "Server not initialized")
                        
            except json.JSONDecodeError:
                # JSONパースエラー
                await self._send_error(writer, None, -32700, "Parse error")
            except Exception as e:
                # 内部エラー
                request_id = request.get("id") if 'request' in locals() else None
                await self._send_error(writer, request_id, -32603, f"Internal error: {str(e)}")
    
    async def _send_response(self, writer, response):
        """レスポンス送信"""
        response_json = json.dumps(response)
        writer.write(response_json.encode('utf-8') + b'\n')
        await writer.drain()
    
    async def _send_error(self, writer, request_id, code, message):
        """エラー送信"""
        error_response = {
            "jsonrpc": "2.0",
            "id": request_id,
            "error": {
                "code": code,
                "message": message
            }
        }
        await self._send_response(writer, error_response)

# 使用例
if __name__ == "__main__":
    mcp = MinimalMCP("Simple Calculator")
    
    @mcp.tool
    async def add(a: int, b: int) -> int:
        """2つの数値を足します"""
        return a + b
    
    @mcp.tool
    def multiply(a: int, b: int) -> int:
        """2つの数値を掛け合わせます"""
        return a * b
    
    asyncio.run(mcp.run())
```

### 注意点

1. **プロトコル仕様への厳密な準拠**: MCP仕様を厳密に守らないと、クライアントとの互換性が失われます。特に初期化フェーズ、エラーコード、メッセージ形式は仕様通りに実装しましょう。

2. **非同期処理の適切な管理**: asyncioを使用する際は、ブロッキング処理を避け、適切なタスク管理を行うことがパフォーマンスと安定性の鍵となります。

3. **エラーハンドリング**: 様々な状況（JSONパースエラー、未知のメソッド、ツール実行エラーなど）に対して適切なエラー応答を返すように実装しましょう。

4. **トランスポート選択**: まずはシンプルなstdioトランスポートから始め、必要に応じてHTTP/SSEトランスポートを追加するのが良いでしょう。

5. **型安全性**: パラメータ検証や結果の型変換は、PydanticなどのライブラリやPythonの型ヒントを活用すると効率的です。

## FastMCPの抽象化技術と設計思想

FastMCPがMCPサーバー開発を簡略化するために採用している設計思想と抽象化技術についてまとめます。

### 「設定より規約」の原則

FastMCPの設計は「設定より規約」(Convention over Configuration)の原則に基づいています。これにより、開発者は最小限の明示的な設定で多くの機能を利用できます。

- **関数名からの名前推論**: ツールやリソースの名前は、関数名から自動的に推論されます。
- **ドキュメント文字列の活用**: 関数のドキュメント文字列が説明として自動的に使用されます。
- **型ヒントからのスキーマ生成**: パラメータの型ヒントからJSONスキーマが自動生成されます。
- **関数署名に基づく機能拡張**: Context型のパラメータがあれば自動的にコンテキストが注入されます。

この規約重視のアプローチにより、開発者は「何をするか」だけを定義し、「どのように実現するか」の詳細をFastMCPに委ねることができます。

### 共通の抽象化技術

FastMCPは以下の主要な抽象化技術を活用しています：

1. **デコレータ**: FastMCPの中核となる抽象化メカニズムです。@mcp.tool、@mcp.resource、@mcp.promptといったデコレータにより、通常のPython関数からMCPコンポーネントへの変換が自動的に行われます。

2. **型ヒントとPydantic**: Pythonの型ヒントは、単なるドキュメントではなく、実行時の引数検証とMCPプロトコルで必要なJSONスキーマ生成に使用されます。FastMCPは内部でPydanticを活用して、強力な型検証と変換を実現しています。

3. **非同期プログラミング(asyncio)**: FastMCPはPythonのasyncioを全面的に活用し、ツール、リソース、ライフサイクル管理などでasync/await構文をサポートしています。これにより、I/O処理を含む長時間実行される関数でもサーバーがブロックされず、高い並行性を実現します。

4. **コンテキスト注入**: Context オブジェクトを引数として注入するパターンは、依存性注入の一形態です。これにより、ハンドラ関数はサーバーの機能（ロギング、進捗報告など）に制御された方法でアクセスでき、ハンドラとサーバー内部との結合が疎になります。

5. **コンテキストマネージャ**: lifespan機能では、Pythonのコンテキストマネージャ（特にasynccontextmanager）を活用して、サーバーの起動から終了までのリソースライフサイクル管理を簡素化しています。

### 抽象化のレイヤー

FastMCPの抽象化は複数のレイヤーで構成されています：

- **プロトコルレイヤー**: MCP仕様に準拠したJSON-RPCメッセージングの詳細を抽象化
- **トランスポートレイヤー**: stdio、HTTP/SSEなどの通信方式の違いを抽象化
- **実行レイヤー**: 同期/非同期関数の違いを吸収し、並行処理を管理
- **スキーマレイヤー**: 関数シグネチャからMCP互換のスキーマを自動生成
- **コンテキストレイヤー**: リクエスト固有の状態や共有リソースへのアクセスを抽象化

これらの抽象化により、開発者はMCPプロトコルの複雑な詳細を気にすることなく、ビジネスロジックの実装に集中できます。FastMCPが公式SDKに統合されたことは、この開発者エクスペリエンス重視のアプローチがMCPコミュニティ全体で支持されていることを示しています。

## コード例（使用方法）

```python
# example_usage.py
import asyncio
from minimal_mcp import MinimalMCP

# サーバーインスタンス作成
mcp = MinimalMCP("My Custom MCP Server")

# ツール定義
@mcp.tool
async def fetch_weather(city: str) -> str:
    """指定された都市の天気を取得します"""
    # 実際には外部APIを呼び出すなどの処理
    await asyncio.sleep(1)  # 非同期処理のシミュレーション
    return f"{city}の天気は晴れです"

@mcp.tool
def calculate_area(width: float, height: float) -> float:
    """長方形の面積を計算します"""
    return width * height

# サーバー起動
if __name__ == "__main__":
    asyncio.run(mcp.run())
```

## 参考リンク
- [MCP公式仕様](https://github.com/llm-tools/model-context-protocol)
- [mcp-python-sdk](https://github.com/modelcontextprotocol/python-sdk)
- [asyncioドキュメント](https://docs.python.org/3/library/asyncio.html)
- [JSON-RPC 2.0仕様](https://www.jsonrpc.org/specification)

## 作成日
2024/04/09

## ステータス
- [x] 下書き
- [ ] レビュー待ち
- [ ] 完了