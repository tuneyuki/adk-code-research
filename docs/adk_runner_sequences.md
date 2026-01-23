# Runner Deep Dive: シーケンス図による動作解析

Runner が具体的にどのように動作しているか、主要なパターン（新規会話開始、ツール実行とイベント処理）をシーケンス図で示します。

## 1. 新規会話の開始 (Start Invocation)

ユーザーが新しいメッセージを送信し、Agent が応答を開始するまでのフローです。

```mermaid
sequenceDiagram
    participant User
    participant Runner
    participant SessionService
    participant Ctx as InvocationContext
    participant Plugins as PluginManager
    participant Agent

    User->>Runner: run_async(user_id, session_id, message="こんにちは")
    activate Runner

    Runner->>SessionService: get_session(session_id)
    SessionService-->>Runner: Session (Existing or None)
    
    note over Runner: セッションが存在しない場合はエラー(要事前作成)

    %% Context Setup
    Runner->>Runner: _setup_context_for_new_invocation()
    activate Runner
    Runner->>Ctx: create()
    
    %% Handle New Message
    Runner->>Plugins: run_on_user_message_callback(message)
    Plugins-->>Runner: modified_message (Optional)
    
    Runner->>SessionService: append_event(UserEvent)
    deactivate Runner

    %% Execution Phase
    Runner->>Runner: _exec_with_plugin()
    activate Runner
    
    Runner->>Plugins: run_before_run_callback()
    
    Runner->>Agent: run_async(ctx)
    activate Agent
    
    loop Event Stream
        Agent-->>Runner: yield Event (Response/ToolCall)
        
        Runner->>SessionService: append_event(Event)
        Runner->>Plugins: run_on_event_callback(Event)
        
        Runner-->>User: yield Event
    end
    
    deactivate Agent
    deactivate Runner
    
    %% Compaction (Optional)
    opt Event Compaction check
        Runner->>SessionService: _run_compaction_for_sliding_window()
    end

    deactivate Runner
```

### ポイント
1.  **前処理**: Agent が動く前に、Runner はセッションの取得、コンテキスト作成、ユーザーメッセージの永続化（`append_event`）を完了させます。
2.  **イベントループ**: Agent が `yield` したイベントは、即座にユーザーに返されるだけでなく、Runner によって**同時に `SessionService` へ保存**されます。これが「Runner を通さないと記憶喪失になる」理由の技術的詳細です。

## 2. ツール実行とイベント処理

Agent がツール（検索など）を呼び出す場合のフローです。ADK ではツール実行結果も `Event` として扱われます。

```mermaid
sequenceDiagram
    participant Agent
    participant Runner
    participant SessionService
    participant Tool as Tool (VertexAiSearchなど)

    note right of Agent: LLMがツール使用を決定
    
    Agent->>Agent: yield Event(FunctionCall)
    Agent-->>Runner: Event(FunctionCall)
    Runner->>SessionService: append_event(Event)
    Runner-->>User: yield Event(FunctionCall)

    %% ツール実行フェーズ (Agent内部またはToolExecutor)
    activate Agent
    Agent->>Tool: execute()
    Tool-->>Agent: Result (Search Results etc.)
    
    Agent->>Agent: yield Event(FunctionResponse)
    Agent-->>Runner: Event(FunctionResponse)
    
    note over Runner: ツール実行結果も履歴に残る
    Runner->>SessionService: append_event(Event)
    Runner-->>User: yield Event(FunctionResponse)

    %% 最終回答生成
    Agent->>Agent: Generate Final Response
    Agent-->>Runner: Event(TextResponse)
    Runner->>SessionService: append_event(Event)
    Runner-->>User: yield Event(TextResponse)
    deactivate Agent
```

### ポイント
1.  **Function Call もイベント**: 「検索したい」という要求 (`FunctionCall`) も、「検索結果」 (`FunctionResponse`) も、すべて `Event` として Runner 経由で保存されます。
2.  **一元管理**: これにより、会話履歴には「ユーザー発言 -> Agent検索要求 -> 検索結果 -> Agent最終回答」という完全なログが残り、次回の会話で文脈として利用可能になります。

## 3. 会話の再開 (Resume Invocation)

中断された処理や、特定の状態からの再開を行う場合です。（`invocation_id` 指定）

```mermaid
sequenceDiagram
    participant User
    participant Runner
    participant SessionService
    participant Ctx as InvocationContext

    User->>Runner: run_async(…, invocation_id="inv-123")
    activate Runner
    
    Runner->>SessionService: get_session()
    
    Runner->>Runner: _setup_context_for_resumed_invocation()
    activate Runner
    
    Runner->>SessionService: 履歴から対象の user_message を検索
    SessionService-->>Runner: user_message
    
    Runner->>Ctx: create(resume=True)
    
    Runner->>Ctx: populate_invocation_agent_states()
    note right of Runner: 前回終了時のAgent状態を復元
    
    deactivate Runner
    
    Runner->>Runner: _exec_with_plugin()
    note over Runner: 以降は新規開始と同じフローで実行再開
```

### ポイント
1.  **状態復元**: Runner はセッション履歴から過去の `AgentState` を読み出し、Agent を「前回の続き」から動かせるようにセットアップします。
