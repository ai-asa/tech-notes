---
title: "FastMCPの実装を理解する：MCPプロトコルの抽象化と内部動作"
emoji: "⚙️"
type: "tech"
topics: ["AI", "MCP", "Claude", "Python", "FastMCP"]
published: false
---

# FastMCPの実装を理解する

皆さん、こんにちは。前回の記事「FastMCPを理解する」では、Model Context Protocol (MCP) とFastMCPの基本概念について解説しました。今回は一歩踏み込んで、FastMCPがどのように低レベルのMCPプロトコルを抽象化し、実装しているかを掘り下げていきます。

## はじめに

FastMCPは、開発者が簡単にMCPサーバーを構築できるように設計された高レベルフレームワークですが、その裏では複雑なプロトコル処理や通信処理が行われています。この記事では、その「魔法」の裏側を探り、FastMCPの内部実装について詳しく解説します。

主に以下のトピックを取り上げます：

- FastMCPのアーキテクチャ設計
- デコレータの内部実装
- リクエスト処理の流れ
- シリアライゼーション/デシリアライゼーション
- トランスポート層の実装
- エラー処理メカニズム

## FastMCPのアーキテクチャ概要

### レイヤー構造

FastMCPは、複数の抽象化レイヤーで構成されています：

```
[ユーザーコード（デコレータ付き関数）]
       ↓
[FastMCP高レベルAPI（デコレータ、FastMCPクラス）]
       ↓
[Python SDK MCP実装（低レベルMCP処理）]
       ↓
[通信層（Stdio/SSE）]
```

この階層構造により、開発者は上位レイヤーの単純化されたAPIのみを扱えば良く、下位レイヤーの複雑な処理から保護されています。

### 主要コンポーネント

FastMCPの実装は以下の主要コンポーネントで構成されています：

1. **FastMCPクラス** - サーバーのエントリーポイントとなるメインクラス
2. **デコレータ実装** - リソース、ツール、プロンプトを登録する機能
3. **リクエストハンドラ** - JSON-RPCリクエストを処理するロジック
4. **トランスポート管理** - 通信プロトコルを実装
5. **コンテキスト機能** - リクエスト処理中のコンテキスト情報を管理

## デコレータの内部実装

### デコレータの仕組み

FastMCPのデコレータ（`@mcp.resource`、`@mcp.tool`、`@mcp.prompt`）は、単なる「シンタックスシュガー」ではなく、関数を登録して変換する重要なメカニズムです。

```python
# FastMCPのリソースデコレータの簡略化された実装例
def resource(uri_template: str):
    def decorator(func):
        # 関数のメタデータを抽出（docstring、パラメータ情報）
        metadata = extract_function_metadata(func)
        
        # 関数をリソースレジストリに登録
        register_resource(uri_template, func, metadata)
        
        # オリジナルの関数をそのまま返す（変更なし）
        return func
    return decorator
```

このデコレータの呼び出しにより、関数はそのままの形で残りますが、内部的にはFastMCP登録システムに追加されます。

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

## まとめ

FastMCPは、MCPプロトコルの複雑さを抽象化し、開発者に分かりやすいAPIを提供するために、いくつもの工夫がなされています：

1. **デコレータによる宣言的なAPI** - 低レベルの登録処理を隠蔽し、簡潔で理解しやすいインターフェースを提供
2. **自動型変換** - Python型とJSON Schema間の双方向変換を自動的に処理
3. **柔軟なトランスポート** - 異なる通信方法をサポートしつつ、一貫したAPIを提供
4. **コンテキスト指向設計** - ハンドラ関数に必要な機能を統一されたContext経由で提供

これらの特徴により、FastMCPは内部実装の複雑さを隠しながら、開発者に強力で使いやすいツールを提供することに成功しています。

FastMCPの内部実装を理解することで、より効果的なMCPサーバーの開発や、問題が発生した際のトラブルシューティングに役立つでしょう。また、この知識は独自のMCPフレームワークを開発する際の参考にもなります。

## 参考リソース

- [MCP仕様](https://github.com/anthropics/anthropic-cookbook/tree/main/mcp/spec)
- [Python SDK ソースコード](https://github.com/modelcontextprotocol/python-sdk)
- [JSON-RPC 2.0仕様](https://www.jsonrpc.org/specification)
- [FastMCP実装の詳細解説](https://github.com/modelcontextprotocol/python-sdk/tree/main/docs) 