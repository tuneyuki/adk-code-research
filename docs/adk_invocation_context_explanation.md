# ADKにおける呼び出しコンテキストと履歴管理

## 概要
本文書では、ADKの `session_service` が会話履歴をどのように管理し、エージェント実行時にコンテキストウィンドウがどのように扱われるかについて説明します。

## セッションサービス (Session Service)
`session_service` は会話履歴の保存と取得を担当します。
- **実装 (Implementations)**:
    - `InMemorySessionService`: イベントをメモリ内に保存します。再起動すると失われます。
    - `SqliteSessionService`: イベントをローカルの SQLite データベースに保存します。
- **保存制限 (Storage Limits)**:
    - どちらの実装も、保存されるイベントの**数**に対してデフォルトの制限を設けていません。手動で管理しない限り、履歴は無制限に増加します。
- **取得 (Retrieval)**:
    - 取得メソッド (`get_session`) は `num_recent_events` (最新のN件) や `after_timestamp` (特定の時間以降) によるフィルタリングをサポートしています。
    - しかし、デフォルトのフロー (`contents.py`) では、現在のブランチの**全**履歴を取得し、「不可視」や不要なイベント（フレームワークイベントなど）のみをフィルタリングします。

## コンテキストウィンドウの扱いと機能の所在
ここでの機能が「ADKのコードで実行されるもの」なのか、「Google Cloud側で実行されるもの」なのかを整理します。

### 1. コンパクション (Compaction) - ADKの機能 (ただし自動化は未実装)
- **機能の所在**: **ADK (クライアントサイド)**
- **ソースコードの事実 (Source Level Facts)**:
    - **消費ロジック**: `google/adk/flows/llm_flows/contents.py` の `_process_compaction_events` 関数に実装されています。履歴内に `compaction` アクションを持つイベントが見つかると、その `start_timestamp` 以前のイベントをフィルタリングして除外します。
    ```python
    # contents.py (抜粋イメージ)
    def _process_compaction_events(events: list[Event]) -> list[Event]:
      # ...
      for event in reversed(events):
        if event.actions and event.actions.compaction:
           # 要約イベントが見つかったら、それより古いイベントをフィルタリング対象にする
           last_compaction_start_time = min(last_compaction_start_time, compaction.start_timestamp)
    ```
    - **生成ロジック**: ADKの標準ライブラリ内 (`google/adk` 配下) 全体を検索しましたが、`EventActions` の `compaction` フィールドに値を設定して新たなイベントを生成するコードは**存在しませんでした**（テストコードや定義部分を除く）。
    - **結論**: 「要約された履歴を正しく処理する（消費する）」機能は実装されていますが、「履歴を要約してイベントを作る（生成する）」機能は、ユーザーが自分で実装する必要があります（スタブコードがあるわけではなく、単に呼び出し側が存在しない状態です）。

### 2. Live API (`run_live`) - Google Cloudの機能
- **機能の所在**: **Google Cloud (Vertex AI / Gemini API)**
- **仕組み**:
    - `RunConfig` で `context_window_compression` を設定すると、ADKはその設定をそのまま Google Cloud の API に渡します。
    - 実際の圧縮処理は、Google Cloud のサーバー側で行われます。
    - **結論**: Google Cloud の機能に依存しています。

### 3. Interactions API (`use_interactions_api=True`) - Google Cloudの機能
- **機能の所在**: **Google Cloud (Vertex AI / Gemini API)**
- **仕組み**:
    - ADKは全履歴を送る代わりに、前回のやり取りを示すID (`previous_interaction_id`) だけを送信します。
    - 会話の履歴や状態（ステート）の管理は、すべて Google Cloud のサーバー側で行われます。
    - **結論**: Google Cloud の機能に依存しています。

## 推奨事項
コンテキストウィンドウの制限（トークン制限）を回避するための現実的なアプローチは以下の通りです：

1.  **Interactions API の使用 (推奨)**:
    - 最も簡単にコンテキストを管理できます。履歴管理を Google Cloud に任せるため、ADK側での複雑な実装が不要です。
    - `Gemini` モデル設定で `use_interactions_api=True` を有効にします。

2.  **Live API の使用**:
    - 音声対話などリアルタイム性が求められる場合はこちらを使用し、必要に応じて圧縮設定を有効にします。

3.  **手動管理 (上級者向け)**:
    - 独自の `SessionService` を実装して古い履歴を削除・アーカイブするか、独自のエージェントを作成して定期的に履歴を要約し、`Compaction` イベントを履歴に挿入するような仕組みを構築します。
