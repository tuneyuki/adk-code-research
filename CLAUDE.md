# CLAUDE.md

## プロジェクト概要

このリポジトリは **google-adk（Agent Development Kit）のソースコード調査ナレッジベース**です。

ADK のソースコードを読み解き、判明した仕様・挙動・アーキテクチャを `docs/` フォルダに Markdown で蓄積していきます。コードを書くことが主目的ではなく、**ドキュメント作成が主要なアウトプット**です。

## ディレクトリ構成

- `docs/` — ADK 調査結果の Markdown ドキュメント群
- `.venv/` — google-adk がインストールされた Python 仮想環境（調査対象のソースコードはここにある）

## 作業の進め方

- ADK のソースコードは `.venv/Lib/site-packages/google/adk/` を直接読む
- バージョンアップ時は差分（diff）を取ってから変更点をドキュメント化する
- 既存ドキュメントのフォーマット・粒度に合わせて新規ドキュメントを作成する
- ドキュメントは日本語で書く

## 現在のADKバージョン

- google-adk: 1.27.2
- 過去の調査: 1.22.1 → 1.25.1 → 1.27.2

## docs/ の主要ドキュメント

### アップデート差分
- `adk_update_1221_to_1251.md` — 1.22.1 → 1.25.1 の変更点
- `adk_update_1251_to_1272.md` — 1.25.1 → 1.27.2 の変更点

### アーキテクチャ解説
- `adk_runner_explanation.md` / `adk_runner_sequences.md` — Runner の仕組み
- `adk_invocation_context_explanation.md` — InvocationContext の解説
- `adk_execution_modes.md` — 実行モード
- `adk_skills_architecture.md` — Skills システムの詳細
- `adk_memory_management_v1_25_0.md` — メモリ管理
- `adk_artifact_explanation.md` — アーティファクト
- `code_executor.md` — コード実行

### セッション・SSE
- `session_persistence_behavior.md` / `session_memory_analysis.md` — セッション永続化
- `sse_session_architecture.md` / `sse_disconnection_mechanism.md` / `sse_disconnect_buffer_behavior.md` — SSE 接続

### Vertex AI Search 関連
- `vertex_ai_search_tool_grounding.md` — グラウンディング
- `usecase_vertex_ai_search_dynamic_config.md` — 動的設定
- `usecase_vertex_ai_search_grounding_metadata.md` — メタデータ検証
- `bigquery-to-vertex-ai-search.md` — BigQuery 連携

### エージェント動作
- `agent_instruction_behavior.md` — instruction の挙動
