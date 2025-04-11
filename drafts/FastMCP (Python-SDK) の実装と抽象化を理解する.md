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

例えば、`@server.tool`デコレータを使用して関数を登録する際の処理シーケンスは以下のように行われます：

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

このシンプルなデコレータの使用により、内部では以下のような複雑な処理が自動的に行われています：

{デコレータの実装について詳細な解説。inspectモジュールの使用方法、関数メタデータの抽出、登録システムへの連携などを説明。実際のデコレータコードと、それが内部でどのように処理されるかの例を示す。}

### 型ヒントからスキーマへの自動変換

FastMCPは、Pythonの型ヒントとdocstringを解析して、MCPプロトコルに必要なJSON Schemaを自動的に生成します。これにより、開発者は複雑なスキーマを手動で定義する必要がなくなります。

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

{型ヒント解析の実装詳細。typingモジュールの使用方法、各種Python型からJSONスキーマへの変換ロジック、特殊型（Optional, Union, List等）の処理方法、Pydanticモデルとの連携などを説明。実際の例を示す。}

### 非同期通信の管理

FastMCPは、asyncioを基盤とした非同期プログラミングモデルを採用し、同期処理と非同期処理を透過的に扱えるようにしています。リクエスト処理は以下のようにルーティングされます：

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

{非同期処理の実装詳細。asyncioの使用方法、同期/非同期関数の自動検出と処理の分岐、イベントループ管理、非同期コンテキスト管理などを説明。コード例を含める。}

### コンテキスト提供

FastMCPは、各リクエスト処理に対して専用のコンテキストオブジェクトを提供し、ログ記録、進捗報告、リソースアクセスなどの機能を簡単に利用できるようにしています。これにより、ハンドラ関数は必要な機能にアクセスしやすくなっています。

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

{コンテキスト機能の実装詳細。Contextクラスの設計、リクエスト間のコンテキスト分離、依存性注入の仕組み、ライフサイクル管理との連携などを説明。実際のContextクラスの使用例を示す。}

### URIベースのリソースシステム

FastMCPは、URIテンプレートを使った直感的なリソース定義と、効率的なパラメータ抽出機能を提供しています。リソースのURIパターンは正規表現に変換され、リクエスト時に効率的にマッチングされます。

```python
def _register_resource_template(self, template: str, func):
    # URIテンプレート（例："users://{user_id}/profile"）を
    # 正規表現パターン（例："users://(?P<user_id>[^/]+)/profile"）に変換
    pattern_str = re.sub(r'\{([^}]+)\}', r'(?P<\1>[^/]+)', template)
    pattern = re.compile(f"^{pattern_str}$")
    
    self._template_resources[template] = (func, pattern)

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
```

{URIシステムの実装詳細。テンプレート解析と正規表現変換、パラメータ抽出ロジック、マッチング優先順位の処理などを説明。URIテンプレートを使った実際の例を示す。}

## lifespan管理の実装

FastMCPは、サーバーの起動時と終了時に共有リソースを効率的に管理するための、lifespanコンテキストマネージャをサポートしています。これにより、データベース接続などの共有リソースをツール間で効率的に管理できます。

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

{lifespan管理の実装詳細。非同期コンテキストマネージャの使用方法、リソース初期化と終了処理の管理、エラーハンドリング、コンテキスト伝播の仕組みなどを説明。実際の使用例を示す。}

## 処理フロー
{リクエスト処理フローの詳細説明。リクエスト受信からレスポンス送信までの各ステップ、コンポーネント間の相互作用、エラー処理の流れなどを説明。シーケンス図やフロー図を用いて視覚的に表現。}

## まとめ

FastMCPは、MCPプロトコルの複雑さを様々なレベルで抽象化し、開発者が本質的な機能実装に集中できるようにします。デコレータ、型ヒント解析、非同期処理、コンテキスト提供などの技術を巧みに組み合わせることで、簡潔で直感的なAPIと強力な内部実装を両立させています。

これらの抽象化により、通常は多くのコードが必要な複雑なMCPサーバーが、少ないコードで実装可能になります。また、独自のMCPサーバーを実装する場合は、FastMCPの抽象化アプローチを参考にすることで、効率的な設計が可能になるでしょう。

## 参考リソース

- [MCP仕様](https://github.com/anthropics/anthropic-cookbook/tree/main/mcp/spec)
- [Python SDK ソースコード](https://github.com/modelcontextprotocol/python-sdk)
- [JSON-RPC 2.0仕様](https://www.jsonrpc.org/specification)
- [FastMCP実装の詳細解説](https://github.com/modelcontextprotocol/python-sdk/tree/main/docs)

