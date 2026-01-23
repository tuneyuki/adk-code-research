# InvocationContext コードレベル解析

`InvocationContext` は、Agent の**1回の実行単位（Invocation）**における「すべて」を保持する巨大なコンテナオブジェクトです。
ソースコード (`google.adk.agents.invocation_context`) に基づき、その役割と主要プロパティを解説します。

## 1. 役割: 実行状態の「中心地」

Runner が `run_async` されたとき、最初に生成されるのがこのオブジェクトです。
Agent やツール、プラグインは、すべてこの Context を通じて必要な情報にアクセスします。

**主な責任:**
1.  **サービスの提供**: DBやファイル操作など、外部連携機能へのアクセス権を提供します。
2.  **実行スコープの定義**: 現在「誰が（User/Agent）」「どのセッションで」「何をしているか」を定義します。
3.  **状態の保持**: 各 Agent のメモリ状態 (`agent_states`) や、終了フラグ (`end_invocation`) を管理します。

## 2. 主要プロパティ (Source Code Based)

クラス定義 (`class InvocationContext(BaseModel)`) から見る主要な構成要素です。

### A. サービス連携 (Services)
Agent が外部とやり取りするためのインターフェース群です。
*   `session_service`: 会話履歴の保存・読み込み。
*   `artifact_service`: 生成ファイル（画像やPDFなど）の保存。
*   `memory_service`: 長期記憶の検索。
*   `plugin_manager`: 拡張機能（フック処理）の管理。

### B. 実行パラメータ (Execution Scope)
*   `invocation_id`: この実行を一意に識別するID（UUID）。再開時にも使用。
*   `session`: 現在のセッションオブジェクト全体。
*   `user_content`: 今回の実行のトリガーとなったユーザーメッセージ。
*   `agent`: 現在実行中の Agent インスタンス。
*   `run_config`: 実行時の設定（LLM呼び出し上限など）。

### C. 実行制御 (Control Flow)
*   `agent_states`: 各 Agent の内部状態を保持する辞書。Resume 時にここから復元されます。
    *   型: `dict[str, dict[str, Any]]` (Agent名をキーとした状態辞書)
*   `end_of_agents`: 各 Agent が「完了」したかどうかを管理するフラグ。
*   `end_invocation`: **重要**。これを `True` にセットすると、Agent のループが強制終了します（プラグイン等から制御可能）。

## 3. 具体的な動作イメージ

`Runner.run_async` 内部での `InvocationContext` のライフサイクルは以下のようになります。

1.  **生成**: `Runner._new_invocation_context()` で初期化されます。ここでサービス類が注入されます。
2.  **伝播**: Agent の `run_async(ctx)` メソッドの引数として渡されます。
3.  **参照・更新**:
    *   Agent は `ctx.session` から過去の会話を読みます。
    *   ツールは `ctx.artifact_service` を使ってファイルを保存します。
    *   `ctx.agent_states` が更新され、ステップごとの思考状態が記録されます。
4.  **破棄**: 実行（Invocation）が完了すると、この Context オブジェクトの役割は終わりますが、変更された内容は `SessionService` を通じて永続化されます。
