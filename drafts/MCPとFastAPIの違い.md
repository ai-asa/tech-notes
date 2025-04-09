# Model Context Protocol(MCP)とFastAPIの違い

# 誰に向けたものか
MCPサーバーの開発を行っているなかで「これAPIでいいんじゃないか？」と思うことが多々あり、MCPとAPIの違いについて備忘録的な整理を行いました。

最近のAI開発の現場では、大規模言語モデル（LLM）と外部ツールを接続するためのプロトコルとして、AnthropicのModel Context Protocol（MCP）が注目を集めています。本記事では、MCPとWeb APIフレームワークであるFastAPIの違いについて解説し、それぞれの適切な使用シーンを考察します。

## 詳細

### 背景：MCPとは何か

Model Context Protocol（MCP）は、AIアプリケーション、特に大規模言語モデル（LLM）が外部のデータソースやツールとどのように対話するかを標準化するために設計されたオープンプロトコルです。Anthropicによって2024年後半に導入され、「AIアプリケーションのためのUSB-Cポート」と例えられています。

従来、M個の異なるLLMやAIシステムをN個の異なるツールやデータソースに接続するには、M * N個のカスタム統合が必要でした。これは非効率的で、スケーリングが困難な手法でした。MCPはこの問題を根本的に単純化し、N + Mの構成へと転換します。一度MCP標準に準拠すれば、どのクライアントもどのサーバーとも相互運用可能になります。

### MCPアーキテクチャ

MCPの公式アーキテクチャは、次の3つの主要コンポーネントで構成されています：

1. **MCP Host（ホスト）**: AI/LLMが常駐し、タスク実行の環境を提供する主要なアプリケーション
   - クライアントのライフサイクル管理や接続許可の制御などを担当
   - 例：Claude Desktop、IDEプラグインなど

2. **MCP Client（クライアント）**: 通常、ホストによって管理される中間プロセス
   - 特定のMCPサーバーとの接続維持やメッセージルーティングを担当
   - サーバーとの1対1の接続を確立

3. **MCP Server（サーバー）**: 特定の機能を提供するためにMCP標準を実装するプログラム
   - 外部ツール、データリソース、またはワークフローへのアクセスを提供
   - リソース、ツール、プロンプトなどの機能を公開

### MCPの通信メカニズム

MCPは、コア通信プロトコルとしてJSON-RPC 2.0を採用しています。メッセージタイプには以下があります：

- **Requests（リクエスト）**: クライアントからサーバーへメソッドを呼び出すメッセージ
- **Responses（レスポンス）**: サーバーからクライアントへの応答メッセージ
- **Notifications（通知）**: 一方向メッセージで、応答を必要としない。進捗更新やリソース変更などのイベントに使用

トランスポート層として、以下がサポートされています：

- **Stdio Transport**: 標準入出力ストリームを使用。ホストがMCPサーバーを直接起動するローカルプロセスに最適
- **HTTP with Server-Sent Events (SSE) Transport**: クライアント→サーバーメッセージにはHTTP POSTを、サーバー→クライアントメッセージ（主に通知）にはSSEを使用

### MCPの主要プリミティブ

MCPは以下の主要なデータ構造（プリミティブ）を定義しています：

1. **Tools（ツール）**: AIが呼び出すことができる実行可能な関数やアクション
   - 計算の実行、API呼び出し、データベースクエリ、ファイル操作など
   - REST APIのPOSTエンドポイントに類似
   - モデル（AI）によって制御される

2. **Resources（リソース）**: AIがコンテキストを豊かにするために読み取れるデータ
   - APIレスポンス、ファイルコンテンツ、データベースレコードなど
   - REST APIのGETエンドポイントに類似
   - アプリケーションによって制御される

3. **Prompts（プロンプト）**: サーバーが提供する準備済みの指示やテンプレート
   - 特定のタスクを達成するための再利用可能なワークフロー
   - ユーザーによって明示的に選択されることが多い

4. **Roots（ルート）**: サーバーのアクセス範囲を定義するエントリポイント

5. **Sampling（サンプリング）**: サーバーがAIに補完生成を要求するメカニズム

### FastMCP: MCPの実装

FastMCPは、PythonでMCPサーバーを構築することを簡素化するための高レベルで使いやすいフレームワークです。主な特徴は以下の通りです：

- 高速な開発: 少ないコード量で迅速に開発できる
- Pythonic: Python開発者にとって自然なインターフェース
- デコレータの活用: 機能定義に@mcp.tool, @mcp.resourceなどのデコレータを使用

FastMCPでは、Contextオブジェクトを活用して、複数の高度な機能を提供しています：

```python
from mcp.server.fastmcp import FastMCP, Context

mcp = FastMCP("ContextExample")

@mcp.tool()
async def process_file(file_uri: str, ctx: Context) -> str:
    # ログ記録
    ctx.info(f"Processing file: {file_uri}")
    
    # 処理ロジック...
    
    # 非同期で進捗報告
    await ctx.report_progress(50, 100)
    
    # 残りの処理...
    await ctx.report_progress(100, 100)
    
    return f"Successfully processed {file_uri}"
```

特筆すべき機能として：

- **ログ記録**: ctx.debug(), ctx.info(), ctx.warning(), ctx.error()メソッドによるログ送信
- **進捗報告**: await ctx.report_progress(current, total)による長時間タスクの進捗通知
- **リソースアクセス**: await ctx.read_resource(uri)によるサーバー内の他リソースへのアクセス
- **非同期処理**: async/awaitを用いた非同期操作のサポート

### FastAPIとの比較

FastAPIは、Pythonで高性能なWeb APIを構築するための最新のフレームワークです。MCPとFastAPIには以下のような違いがあります：

#### 状態管理
- **MCP**: 接続はステートフル（セッションベース）。サーバーはセッション内で状態を維持可能。
- **FastAPI/REST API**: 主にステートレス。各リクエストは自己完結型で、状態はクライアント側で管理が基本。

#### 通信方向性
- **MCP**: 双方向（クライアント↔サーバーリクエスト/レスポンス、サーバー→クライアント通知）。サーバーは進捗報告などの通知を非同期で送信可能。
- **FastAPI/REST API**: 主に一方向（クライアント→サーバーリクエスト、サーバー→クライアントレスポンス）。WebSocketを追加実装することで双方向も可能だが、標準機能ではない。

#### 主要な焦点
- **MCP**: プロシージャ呼び出し（Tools）、AIコンテキスト提供（Resources、Prompts）に特化。LLMとの連携を前提とした設計。
- **FastAPI/REST API**: リソース状態の表現と転送（CRUD操作）、一般的なWeb APIの機能提供。汎用的なクライアントとの連携を想定。

#### プロトコル
- **MCP**: JSON-RPC 2.0を基盤としている。
- **FastAPI**: RESTful原則に基づくHTTP/S上のアーキテクチャスタイル。

#### 非同期処理
- **MCP**: サーバーからの一方向通知（進捗報告など）が標準でサポート。長時間タスクの進捗をリアルタイムに報告可能。
- **FastAPI**: 標準REST APIでは非同期レスポンスはサポートされず、WebSocketか他の追加実装が必要。

#### 典型的な用途
- **MCP**: AIエージェント統合、対話型ツール、複雑なワークフロー、リアルタイム更新。
- **FastAPI**: 汎用的なWeb API、マイクロサービス、モバイルバックエンド、リソース指向サービス。

### MCPとFastAPIはどちらを選ぶべきか

以下のような場合にMCPが適しています：
- AIエージェント（特にLLM）と外部ツールの統合が主目的
- 双方向かつリアルタイムな通信が必要
- 複雑なマルチステップのAIインタラクションをサポートしたい
- ステートフルな接続や状態管理が重要
- 処理の進捗をリアルタイムに報告する必要がある
- AIが外部ツールを使用する標準化されたインターフェースが欲しい

以下のような場合にFastAPIが適しています：
- 汎用的なWeb APIの構築
- パフォーマンスが最優先（FastAPIは非常に高速）
- RESTfulなリソース指向のAPIが必要
- 広範なクライアント（ブラウザ、モバイルアプリなど）との互換性が必要
- OpenAPIによる自動ドキュメント生成やバリデーションが欲しい

## コード例

### MCPサーバーのシンプルな例（FastMCPを使用）

```python
from mcp.server.fastmcp import FastMCP, Context

# サーバーインスタンスを作成
mcp = FastMCP("ResearchTools")

# リソースの定義
@mcp.resource("papers://{paper_id}")
def read_paper(paper_id: str) -> str:
    """指定されたIDの論文の内容を取得します"""
    # 論文データの取得ロジックがここに入る
    return f"This is the content of paper {paper_id}..."

# ツールの定義
@mcp.tool()
async def search_papers(query: str, max_results: int = 5, ctx: Context) -> dict:
    """
    論文を検索するツール
    """
    ctx.info(f"Searching for papers about: {query}")
    
    # 検索の実装 (実際はAPIやデータベースにアクセスする)
    # 長時間処理を想定して進捗報告
    await ctx.report_progress(25, 100)
    papers = [
        {"id": "2401.12345", "title": f"Example Paper 1 about {query}"},
        {"id": "2401.67890", "title": f"Example Paper 2 about {query}"},
    ]
    await ctx.report_progress(100, 100)
    
    return {
        "papers": papers
    }

# サーバーの起動
if __name__ == "__main__":
    mcp.run()
```

### 同様の機能をFastAPIで実装した例

```python
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel
from typing import List, Optional
import time

app = FastAPI()

# 進捗状況を保存するための簡易的なストレージ（本番では適切なストレージを使用すべき）
progress_store = {}

class Paper(BaseModel):
    id: str
    title: str

class PaperSearchResponse(BaseModel):
    papers: List[Paper]
    task_id: Optional[str] = None

class PaperContent(BaseModel):
    uri: str
    content: str
    mime_type: str

class ProgressResponse(BaseModel):
    current: int
    total: int
    task_id: str

# 長時間のタスクを模擬
def background_search(task_id: str, query: str):
    progress_store[task_id] = {"current": 0, "total": 100}
    # 25%進捗
    time.sleep(1)
    progress_store[task_id] = {"current": 25, "total": 100}
    # 残りの処理
    time.sleep(2)
    progress_store[task_id] = {"current": 100, "total": 100}

@app.get("/papers/search", response_model=PaperSearchResponse)
async def search_papers(query: str, max_results: int = 5, background_task: bool = False):
    """
    論文を検索するAPI
    background_task=Trueの場合はバックグラウンドで処理し、進捗を別エンドポイントで確認可能
    """
    if background_task:
        # 非同期処理のためのタスクID
        task_id = f"search_{int(time.time())}"
        background_tasks = BackgroundTasks()
        background_tasks.add_task(background_search, task_id, query)
        # 即時レスポンスを返し、処理は裏で継続
        return {
            "papers": [],  # 初期は空
            "task_id": task_id
        }
    else:
        # 同期的に処理して結果を返す
        return {
            "papers": [
                {"id": "2401.12345", "title": f"Example Paper 1 about {query}"},
                {"id": "2401.67890", "title": f"Example Paper 2 about {query}"},
            ]
        }

@app.get("/papers/{paper_id}", response_model=PaperContent)
async def read_paper(paper_id: str):
    """
    論文の内容を取得するAPI
    """
    return {
        "uri": f"https://api.example.com/papers/{paper_id}",
        "content": f"This is the content of paper {paper_id}...",
        "mime_type": "text/plain"
    }

@app.get("/tasks/{task_id}/progress", response_model=ProgressResponse)
async def get_progress(task_id: str):
    """
    タスクの進捗状況を確認するAPI
    """
    if task_id not in progress_store:
        return {"current": 0, "total": 100, "task_id": task_id}
    
    progress = progress_store[task_id]
    return {
        "current": progress["current"],
        "total": progress["total"],
        "task_id": task_id
    }
```

## まとめ

MCPとFastAPIは、異なる目的のために設計された異なるプロトコル/フレームワークです：

- **MCP**はAIエージェントと外部ツールの統合に特化した双方向プロトコルで、ステートフル接続、双方向通信、進捗報告、AIコンテキスト提供などの特徴があります。特にFastMCPを使うと、Pythonで簡潔かつ効率的にMCPサーバーを実装できます。

- **FastAPI**は汎用的なWeb API開発のための高速なフレームワークで、RESTful原則、優れたパフォーマンス、自動ドキュメント生成などの特徴があります。進捗報告のような非同期通知は標準機能ではなく、追加実装が必要です。

どちらが「より優れている」ということではなく、特定の用途に対してどちらがより適しているかという問題です。AIエージェントの開発に焦点を当てる場合はMCPが、より広範なクライアントとの互換性や汎用的なAPIが必要な場合はFastAPIが適しているでしょう。

場合によっては、MCPサーバーの内部実装にFastAPIを使用することも可能です。MCPはプロトコルであり、その実装にはさまざまなフレームワークが使用できるからです。

## 参考リンク
- [Anthropic MCP公式ドキュメント](https://docs.anthropic.com/mcp/overview)
- [FastMCP公式リポジトリ](https://github.com/modelcontextprotocol/python-sdk)
- [FastAPI公式ドキュメント](https://fastapi.tiangolo.com/)
- [JSON-RPC 2.0仕様](https://www.jsonrpc.org/specification)

## 作成日
2024/04/09

## ステータス
- [x] 下書き
- [ ] レビュー待ち
- [ ] 完了 