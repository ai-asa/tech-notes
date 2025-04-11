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
{JSON-RPC 2.0の説明、実際のデータスキーマ例}
{stdioの説明、実際の例}
### SSE
{SSEの解説}
### ASGI
{ASGIの解説}

## MCPの主要な実装
### 構成
{MCPの実装において必要な要素、機能を整理}
### 簡単な実装
{FastMCPを使わずにMCPサーバーを構築するうえでの最小限の簡単な実装コード}

# FastMCPの実装について
## 抽象化レイヤー
FastMCPは以下のような抽象化レイヤー構造を持っています。これによって、**開発者はAPIレイヤーのみを理解すればデータスキーマや通信プロトコルについて意識せずにMCPサーバーを構築することが可能**であり、機能開発に注力することができます。

|深度| 抽象化レイヤー | 役割 | 提供する機能 |
|----|--------------|------|-----------------|
| 浅 | API | MCPサーバー実装を簡潔で直感的に記述 | FastMCPクラス、デコレータ (@tool, @resource, @prompt) など|
| ↓ | 機能マネージャー | 特定の機能ドメインの管理と抽象化 | ToolManager, ResourceManager, PromptManager など |
| ↓ | データモデリング | 型安全なデータ構造と検証 | Pydanticモデル、検証ロジック、JSONスキーマ |
| ↓ | MCPコア | MCP仕様の実装と基本機能 | MCPServer, RequestContext, リクエスト/レスポンス処理 |
| 深 | トランスポート | 通信方式の実装 | SseServerTransport, stdio_server, ASGIインテグレーション |

## 主要コンポーネント

FastMCPの実装は以下の主要コンポーネントで構成されています：

1. **FastMCPクラス** - サーバーのエントリーポイントとなるメインクラス
2. **デコレータ** - リソース、ツール、プロンプトを登録する機能
3. **リクエストハンドラ** - JSON-RPCリクエストを処理するロジック
4. **トランスポート管理** - 通信プロトコルを実装
5. **コンテキスト機能** - リクエスト処理中のコンテキスト情報を管理

すべての実装を理解するのは大変なので、ここでは一部の重要なクラスやデコレータの実装を理解するに留めて、大まかな実装の設計や方針を概観することを目指したいと思います。
:::message
本記事で触れるFastMCPの実装は主に、MCPサーバーのエントリーポイントとなるFastMCPクラスと関連するコンポーネントです。その他の実装についても気になる方は、実際にリポジトリを確認することをお勧めします
:::
### デコレータの実装
FastMCPのデコレータ（`@server.resource`、`@server.tool`、`@server.prompt`）は、単なるシンタックスシュガーではなく、**デコレーションした関数を登録し、AIがJSON-RPCリクエストを通じて実行できるようにします。**

例えば、`@server.tool`デコレータを使用して関数を登録する際の処理シーケンスは以下のように行われます。
sequenceDiagram
    participant User as ユーザーコード
    participant FastMCP
    participant ToolManager
    participant Tool

    User->>FastMCP: @server.tool()デコレータを使用
    FastMCP->>ToolManager: add_tool(fn, name, description)
    ToolManager->>Tool: Tool.from_function(fn)
    Note over Tool: 引数のJSONスキーマ生成
    Tool-->>ToolManager: Toolインスタンス
    ToolManager->>ToolManager: self._tools[name] = tool
    ToolManager-->>FastMCP: toolインスタンス
    FastMCP-->>User: デコレート済み関数

例えば、以下のように`@server.tool`デコレータを呼び出したとします。

```python
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("Demo Server")

@mcp.tool("add")
def add(a: int, b: int) -> int:
    return a + b

if __name__ == "__main__":
    # SSEトランスポートを使用して起動
    mcp.run("sse")
```



このように、デコレータの呼び出しにより関数はFastMCPクラスの_tool辞書に登録されます。

### 型ヒントとdocstringの処理

FastMCPは、関数の型ヒントとdocstringを解析して、MCP仕様に準拠したメタデータを生成します：

```python
def extract_function_metadata(func):
    # インスペクション機能を使用して関数シグネチャを解析
    signature = inspect.signature(func)
    
    # docstringを取得
    doc = inspect.getdoc(func) or ""
    
    # パラメータと型ヒントを抽出
    parameters = {}
    for name, param in signature.parameters.items():
        # Contextパラメータは特別扱い
        if str(param.annotation) == "<class 'mcp.server.fastmcp.context.Context'>":
            continue
            
        # 型情報をJSON Schema形式に変換
        param_type = param.annotation
        schema = convert_type_to_json_schema(param_type)
        parameters[name] = schema
    
    return {
        "description": doc,
        "parameters": parameters,
        "return_type": convert_type_to_json_schema(signature.return_annotation)
    }
```

この処理により、Pythonの型ヒント（`int`、`str`、Pydanticモデルなど）は、MCPクライアントが理解できるJSON Schemaに変換されます。

## リクエスト処理の内部フロー

### JSON-RPCリクエストのルーティング

MCPはJSON-RPC 2.0プロトコルをベースにしています。FastMCPがリクエストを受け取ると、以下のような処理が行われます：

```python
async def handle_request(self, request_data: dict):
    # リクエストのメソッドを取得
    method = request_data.get("method")
    
    if method.startswith("resources/"):
        # リソースリクエストの処理
        return await self._handle_resource_request(request_data)
        
    elif method.startswith("tools/"):
        # ツールリクエストの処理
        return await self._handle_tool_request(request_data)
        
    elif method.startswith("prompts/"):
        # プロンプトリクエストの処理
        return await self._handle_prompt_request(request_data)
    
    # その他のMCPメソッド（info、list等）
    return await self._handle_utility_request(request_data)
```

このルーティングにより、リクエストは適切なハンドラに転送されます。

### リソースリクエストの処理例

リソースリクエストの処理は、URIの解析と関数の呼び出しを伴います：

```python
async def _handle_resource_request(self, request_data: dict):
    # URIを取得
    uri = request_data.get("params", {}).get("uri", "")
    
    # URIマッチングを行い、対応する関数とパラメータを抽出
    handler_func, params = self._match_resource_uri(uri)
    
    if not handler_func:
        return {"error": {"code": -32601, "message": f"Resource not found: {uri}"}}
    
    try:
        # 関数を呼び出し
        context = Context(request_data, self)
        
        # 非同期関数かどうかを判定
        if inspect.iscoroutinefunction(handler_func):
            result = await handler_func(**params, ctx=context)
        else:
            result = handler_func(**params)
        
        # 結果を適切な形式に変換
        serialized_result = self._serialize_resource_response(result)
        
        return {
            "result": serialized_result
        }
    except Exception as e:
        # エラー処理
        return {
            "error": {
                "code": -32603,
                "message": str(e)
            }
        }
```

この処理により、URIから適切な関数を見つけ、パラメータを抽出して実行し、結果を適切な形式でクライアントに返します。

## シリアライゼーション/デシリアライゼーション

### データ型の変換

FastMCPは、PythonオブジェクトとJSON間の変換を自動的に処理します：

```python
def _serialize_resource_response(self, result):
    # 文字列の場合
    if isinstance(result, str):
        return {
            "contentType": "text/plain",
            "content": result
        }
    
    # バイナリデータの場合（例：画像）
    elif isinstance(result, bytes) or isinstance(result, Image):
        if isinstance(result, Image):
            # Imageオブジェクトからバイト列とコンテンツタイプを抽出
            content = result.data
            content_type = f"image/{result.format}"
        else:
            content = result
            content_type = "application/octet-stream"
        
        # Base64エンコーディング
        return {
            "contentType": content_type,
            "content": base64.b64encode(content).decode("ascii"),
            "encoding": "base64"
        }
    
    # その他のオブジェクト（辞書、リストなど）
    else:
        # JSONシリアライズ可能なオブジェクトに変換
        return {
            "contentType": "application/json",
            "content": json.dumps(result)
        }
```

この変換処理により、様々なPythonデータ型を適切なMCPレスポンス形式に変換できます。

### URIテンプレートの処理

動的なリソースURIの処理は、内部的にはパターンマッチングを使用しています：

```python
def _match_resource_uri(self, uri: str):
    # 静的URIでの完全一致を最初に試みる
    if uri in self._static_resources:
        return self._static_resources[uri], {}
    
    # テンプレートURIのマッチングを試みる
    for template, (func, pattern) in self._template_resources.items():
        match = pattern.match(uri)
        if match:
            # パターンにマッチした場合、パラメータを抽出
            params = match.groupdict()
            return func, params
    
    return None, {}

def _register_resource_template(self, template: str, func):
    # URIテンプレート（例："users://{user_id}/profile"）を
    # 正規表現パターン（例："users://(?P<user_id>[^/]+)/profile"）に変換
    pattern_str = re.sub(r'\{([^}]+)\}', r'(?P<\1>[^/]+)', template)
    pattern = re.compile(f"^{pattern_str}$")
    
    self._template_resources[template] = (func, pattern)
```

この実装により、`{name}`のようなパラメータを含むURIテンプレートを正規表現に変換し、実際のURIからパラメータを抽出できます。

## トランスポート層の実装

### Stdioトランスポート

標準入出力を使用したトランスポートは、単純ながら効果的です：

```python
async def run_stdio_transport(self):
    """標準入出力を使用してサーバーを実行"""
    # 非同期I/Oのセットアップ
    reader = asyncio.StreamReader()
    protocol = asyncio.StreamReaderProtocol(reader)
    
    loop = asyncio.get_event_loop()
    await loop.connect_read_pipe(lambda: protocol, sys.stdin)
    
    w_transport, w_protocol = await loop.connect_write_pipe(
        asyncio.streams.FlowControlMixin, sys.stdout)
    writer = asyncio.StreamWriter(w_transport, w_protocol, None, loop)
    
    while True:
        # ヘッダー（Content-Length）の読み取り
        header = await reader.readline()
        if not header:
            break
            
        if header.startswith(b'Content-Length: '):
            content_length = int(header.decode().split(': ')[1])
            
            # 空行（ヘッダーとボディの区切り）を読み飛ばす
            await reader.readline()
            
            # 指定された長さのJSONボディを読み取る
            body = await reader.readexactly(content_length)
            request_data = json.loads(body)
            
            # リクエストを処理
            response_data = await self.handle_request(request_data)
            
            # レスポンスをJSONに変換
            response_body = json.dumps(response_data).encode('utf-8')
            
            # ヘッダーとボディを書き込む
            writer.write(f'Content-Length: {len(response_body)}\r\n\r\n'.encode('utf-8'))
            writer.write(response_body)
            await writer.drain()
```

このコードは、標準入出力ストリームを通じて、Content-Lengthヘッダーに基づいたJSON-RPCメッセージの送受信を実装しています。

### SSE (Server-Sent Events) トランスポート

Web対応のSSEトランスポートは、より複雑ですが、ネットワーク経由のアクセスを可能にします：

```python
def create_sse_app(self):
    """ASGI (Starlette) アプリケーションを作成"""
    from starlette.applications import Starlette
    from starlette.routing import Route
    from starlette.responses import Response, JSONResponse
    
    async def handle_rpc(request):
        # POSTリクエストのJSONボディを読み取る
        request_data = await request.json()
        
        # リクエスト処理
        response_data = await self.handle_request(request_data)
        
        # JSONレスポンスを返す
        return JSONResponse(response_data)
    
    async def handle_sse(request):
        # SSE接続のセットアップ
        async def event_generator():
            # クライアントIDの生成
            client_id = str(uuid.uuid4())
            
            # 接続確立メッセージの送信
            yield f"event: connect\ndata: {json.dumps({'client_id': client_id})}\n\n"
            
            # クライアントをイベントリスナーとして登録
            queue = asyncio.Queue()
            self._sse_clients[client_id] = queue
            
            try:
                while True:
                    # キューからイベントを取得
                    event_data = await queue.get()
                    
                    # SSEフォーマットでイベントを送信
                    event_type = event_data.get("event", "message")
                    data = json.dumps(event_data.get("data", {}))
                    yield f"event: {event_type}\ndata: {data}\n\n"
            finally:
                # 接続終了時にクライアントを削除
                if client_id in self._sse_clients:
                    del self._sse_clients[client_id]
        
        # SSEレスポンスを設定
        return Response(
            content=event_generator(),
            media_type="text/event-stream"
        )
    
    # ルートの設定
    routes = [
        Route("/rpc", endpoint=handle_rpc, methods=["POST"]),
        Route("/events", endpoint=handle_sse, methods=["GET"])
    ]
    
    return Starlette(routes=routes)
```

SSEトランスポートでは、クライアント-サーバー間の双方向通信のために、HTTPエンドポイント（`/rpc`）とイベントストリーム（`/events`）の両方を提供します。

## エラー処理メカニズム

### 例外処理とMCPエラー変換

Pythonの例外をMCPエラーレスポンスに変換する処理は以下のようになります：

```python
def _convert_exception_to_error(self, e: Exception):
    """Pythonの例外をMCP JSON-RPCエラーレスポンスに変換"""
    # 標準のJSON-RPCエラーコード
    error_code = -32603  # Internal error
    
    # 特定の例外タイプに応じたエラーコードのマッピング
    if isinstance(e, ValueError):
        # 値に関するエラー (クライアントエラー)
        error_code = -32602  # Invalid params
    elif isinstance(e, PermissionError):
        # 権限に関するエラー
        error_code = -32001  # カスタムエラーコード
    elif isinstance(e, FileNotFoundError):
        # ファイル不在エラー
        error_code = -32002  # カスタムエラーコード
    
    # エラーレスポンスの構築
    return {
        "error": {
            "code": error_code,
            "message": str(e),
            "data": {
                "exception_type": e.__class__.__name__
            }
        }
    }
```

この変換処理により、Pythonの例外が発生した場合でも、クライアントは標準化されたエラー情報を受け取ることができます。

### エラーロギングとデバッグ支援

FastMCPは、エラーを効果的に診断するためのロギング機能を提供します：

```python
def _log_exception(self, e: Exception, request_data: dict):
    """例外とリクエスト情報をログに記録"""
    method = request_data.get("method", "unknown")
    request_id = request_data.get("id", "unknown")
    
    # 詳細なエラー情報をログに記録
    traceback_str = traceback.format_exc()
    
    self.logger.error(
        f"Error processing request {request_id} for method {method}: {str(e)}\n"
        f"Request data: {json.dumps(request_data)}\n"
        f"Traceback: {traceback_str}"
    )
```

このロギングにより、開発者は発生した問題をより効果的にトラブルシューティングできます。

## Contextオブジェクトの実装

### Contextクラスの内部構造

Contextオブジェクトは、ハンドラ関数とFastMCPのコア機能を橋渡しする重要な役割を果たします：

```python
class Context:
    def __init__(self, request_data: dict, server_instance):
        # リクエストデータの保存
        self.request_data = request_data
        self.request_id = request_data.get("id")
        
        # サーバーインスタンスへの参照
        self._server = server_instance
        
        # lifespan_contextがあれば追加
        self.request_context = RequestContext(
            lifespan_context=getattr(server_instance, "_lifespan_context", {})
        )
    
    # ログメソッド
    def debug(self, message: str, data: Optional[dict] = None):
        self._log("debug", message, data)
    
    def info(self, message: str, data: Optional[dict] = None):
        self._log("info", message, data)
    
    def warning(self, message: str, data: Optional[dict] = None):
        self._log("warning", message, data)
    
    def error(self, message: str, data: Optional[dict] = None):
        self._log("error", message, data)
    
    # 内部ログ処理
    def _log(self, level: str, message: str, data: Optional[dict] = None):
        log_data = {
            "level": level,
            "message": message
        }
        if data:
            log_data["data"] = data
            
        # サーバーのログコールバックを呼び出す
        self._server._emit_log(log_data, self.request_id)
    
    # 進捗報告
    async def report_progress(self, current: int, total: int):
        # 進捗情報をクライアントに送信
        progress_data = {
            "current": current,
            "total": total
        }
        await self._server._emit_progress(progress_data, self.request_id)
    
    # リソース読み取り
    async def read_resource(self, uri: str):
        # サーバー内の別のリソースを読み取る
        return await self._server._read_resource_internal(uri)
```

このContextクラスは、ログ記録、進捗報告、リソースアクセスなどの機能を、統一されたインターフェースを通じて提供します。

## lifespan管理の実装

### サーバーライフサイクル管理

FastMCPは、サーバーの起動時と終了時のリソース管理のために、lifespanコンテキストマネージャをサポートしています：

```python
async def _setup_lifespan(self, lifespan):
    """lifespanコンテキストマネージャのセットアップ"""
    if lifespan is None:
        # lifespanが指定されていない場合は空のコンテキストを使用
        self._lifespan_context = {}
        return
        
    try:
        # lifespanコンテキストマネージャを開始
        self._lifespan_cm = lifespan()
        self._lifespan_context = await self._lifespan_cm.__aenter__()
    except Exception as e:
        self.logger.error(f"Error setting up lifespan: {str(e)}")
        self._lifespan_context = {}

async def _cleanup_lifespan(self):
    """lifespanコンテキストマネージャのクリーンアップ"""
    if hasattr(self, "_lifespan_cm"):
        try:
            await self._lifespan_cm.__aexit__(None, None, None)
        except Exception as e:
            self.logger.error(f"Error cleaning up lifespan: {str(e)}")
```

この実装により、データベース接続のような共有リソースを効率的に管理できます。

## 処理フロー
{クライアントとMCPサーバーが通信を開始し、ツールを実行するまでの処理フロー}

