# ADK調査用リポジトリ

**使用 ADK バージョン**: `google-adk==1.22.1`

Google ADK (Agent Development Kit) の挙動確認や仕様調査を行うための検証用リポジトリです。

主な活動内容:
*   **ドキュメント作成**: 調査結果を `docs/` 配下にまとめています（**本リポジトリの主目的**）
*   **コンポーネント調査**: `CodeExecutor`, `Runner`, `VertexAiSearchTool` などの内部挙動の確認
*   **検証コード**: SDK の動作を確認するためのスクリプト群

## ドキュメント一覧

| ファイル | 概要 |
| :--- | :--- |
| [adk_artifact_explanation.md](docs/adk_artifact_explanation.md) | AgentやToolが生成するファイル（Artifact）の仕組みと保存フローの解説 |
| [adk_execution_modes.md](docs/adk_execution_modes.md) | `adk api_server`, `adk web`, `adk run` 各モードの技術スタックと挙動の違い |
| [adk_invocation_context_explanation.md](docs/adk_invocation_context_explanation.md) | 1回の実行単位（Invocation）を管理する `InvocationContext` の詳細 |
| [adk_runner_explanation.md](docs/adk_runner_explanation.md) | Agentのランタイムである `Runner` の役割と必要性 |
| [adk_runner_sequences.md](docs/adk_runner_sequences.md) | Runnerの動作（新規開始、ツール実行、再開）を示すシーケンス図 |
| [code_executor.md](docs/code_executor.md) | コード実行コンポーネント `CodeExecutor` の概要と実行フロー詳細 |
| [session_memory_analysis.md](docs/session_memory_analysis.md) | SSEストリーミング中に切断された場合の SessionMemory の状態分析 |
| [sse_disconnect_buffer_behavior.md](docs/sse_disconnect_buffer_behavior.md) | クライアント切断時にバッファ内の中間テキストが消失するメカニズムの解説 |
| [sse_disconnection_mechanism.md](docs/sse_disconnection_mechanism.md) | Uvicorn/Starlette から ADK に至る切断検知とタスクキャンセルの詳細 |
| [sse_session_architecture.md](docs/sse_session_architecture.md) | SSE と `StreamingResponseAggregator` による中間状態バッファリングのアーキテクチャ |
| [vertex_ai_search_tool_grounding.md](docs/vertex_ai_search_tool_grounding.md) | `VertexAiSearchTool` の戻り値 (`grounding_metadata`) の構造解説 |
