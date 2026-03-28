# google-adk パッケージアップデート内容 (1.25.1 → 1.27.2)

## 概要

`google-adk` パッケージをバージョン `1.25.1` から `1.27.2` にアップデートした際の主要な変更点をまとめます。
（調査対象バージョン： `1.26.0`、`1.27.0`、`1.27.1`、`1.27.2`）

主な変更点は以下の領域に分類されます。

---

## 1. コア機能・アーキテクチャの変更

### 1.1 Durable Runtime サポート（Resumability の強化）

* `Runner._run_async_impl()` のinvocation再開ロジックが大幅にリファクタリングされました。
  * 以前: `invocation_id` の有無で明確に分岐（新規 or 再開）
  * 現在: `is_resumable` フラグで統一的に判定。`new_message` が `FunctionResponse` の場合、自動的に対応する `invocation_id` を解決する `_resolve_invocation_id()` メソッドが追加されました。
* `GetSessionConfig` が `Runner` から `session_service.get_session()` にパススルーされるようになり、セッション取得時のイベントフィルタリングが可能になりました。
* セッション未検出時の例外が `ValueError` から新設の `SessionNotFoundError`（`ValueError` を継承）に変更されました。

### 1.2 トークンコンパクション（リクエスト前圧縮）

* **新規ファイル**: `flows/llm_flows/compaction.py`
  * `CompactionRequestProcessor` が追加され、LLMリクエスト前にセッションイベントのトークン閾値ベースのコンパクションを実行します。
* `SingleFlow` のリクエストプロセッサチェーンにおいて、`compaction` プロセッサが `contents` プロセッサの **前** に実行されるよう配置されています。これにより、コンパクション済みイベントがモデルリクエストのコンテキストに反映されます。
* `InvocationContext` に `events_compaction_config` と `token_compaction_checked` フィールドが追加されました。

### 1.3 プラットフォーム抽象化モジュール

* **新規ディレクトリ**: `platform/`
  * `platform/time.py`: `ContextVar` ベースのタイムプロバイダ抽象化。`set_time_provider()` でカスタムタイムソースを注入可能。テスト時の時間制御に有用。
  * `platform/uuid.py`: `ContextVar` ベースのIDプロバイダ抽象化。`set_id_provider()` でカスタムID生成を注入可能。
* 従来 `datetime.now()` や `uuid.uuid4()` を直接呼んでいた箇所（`Event`, `InvocationContext` 等）が、すべてこのプラットフォームモジュール経由に変更されました。

### 1.4 BaseAgent の変更

* `version` フィールドが追加されました。テレメトリでエージェントのバージョンを識別するために使用されます。
* `parent_agent` フィールドに `exclude=True` が追加され、シリアライズ時に除外されるようになりました。
* `@experimental` デコレータが `@experimental(FeatureName.AGENT_CONFIG)` のように、具体的な `FeatureName` を引数に取る形式に変更されました。

### 1.5 LlmAgent の変更

* **`output_schema` の型拡張**: `type[BaseModel]` のみだった型が `SchemaType` に拡張され、以下をサポート：
  * `type[BaseModel]`、`list[type[BaseModel]]`、`list[primitive]`（`list[str]` 等）、`dict`（生のdictスキーマ）、`Schema`（Google の Schema 型）
* **`canonical_tools()` の並列化**: ツール解決処理が `asyncio.gather()` を使用した並列実行に変更されました。
* `output_key` ハンドリングのリファクタリング: テキストレスポンスのバリデーションが `validate_schema()` ユーティリティに委譲されました。

### 1.6 新規 `Context` クラス

* **新規ファイル**: `agents/context.py`
  * `ReadonlyContext` を継承した新しい `Context` クラスが追加されました。
  * `ToolContext` の実装が `tool_context.py` からこのファイルに移動されました。
  * `tool_context.py` は後方互換のための再エクスポートのみになりました。

---

## 2. ツール (Tools) の拡張

### 2.1 BashTool（新規）

* **新規ファイル**: `tools/bash_tool.py`
  * `ExecuteBashTool`: Skills 内でbashコマンドを実行するためのツール。
  * `BashToolPolicy`: 許可するコマンドのプレフィックスを制限する設定。デフォルトは全コマンド許可（`("*",)`）。
  * ワークスペースディレクトリ内でコマンドを実行し、`subprocess` を使用。

### 2.2 SkillToolset の大幅強化

* **`ListSkillsTool`** が新規追加され、利用可能なスキル一覧をツールとして返せるようになりました。
* **`LoadSkillTool`**: スキル読み込み時に `_adk_activated_skill_{agent_name}` としてステートに記録されるようになりました。
* **`LoadSkillResourceTool`**: `scripts/` ディレクトリからの読み込みをサポート。バイナリファイル検出時のコンテンツインジェクション機能が追加されました。
* **`RunSkillScriptTool`**: スキルの `scripts/` ディレクトリ内のスクリプトを実行するツールが新規追加。
* スキルの `additional_tools` フィールドでツールセットを指定可能に。ADK ツール（FunctionTool 等）もスキル内で使用可能に。
* GCS ファイルシステムサポート: テキストおよび PDF ファイルをスキルリソースとして GCS から読み込み可能に。

### 2.3 FunctionTool の改善

* `tool_context` パラメータ名のハードコードが廃止され、**型アノテーションベースの検出** に変更されました。
  * `find_context_parameter()` ユーティリティが `ToolContext` 型のパラメータを自動検出します。
  * フォールバックとして従来の `'tool_context'` 名が使用されます。

### 2.4 Toolset 認証の統合

* `base_llm_flow.py` に `_resolve_toolset_auth()` が追加されました。
  * ツール一覧取得前に、各ツールセットの `get_auth_config()` をチェックし、認証情報が必要な場合は認証リクエストイベントを生成してinvocationを中断します。

### 2.5 BigQuery / Bigtable ツール

* **新規**: `tools/bigquery/search_tool.py` — Dataplex Catalog 検索ツール。
* BigQuery の設定にジョブラベル制限と内部予約プレフィックスが追加。
* Bigtable: `execute_sql` パラメータサポート、クラスターメタデータツールが追加。非同期関数に変換。

### 2.6 OpenAPI ツール

* `OpenAPIToolset` に `preserve_property_names` オプションが追加され、プロパティ名の変換を抑制可能に。
* `ServiceAccountExchanger` に `id_token` サポートが追加。

---

## 3. セッション管理の改善

### 3.1 temp-scoped ステートの可視性修正

* **重要なバグ修正**: `BaseSessionService.append_event()` で temp ステート（`temp:` プレフィックス）がインメモリセッションに即座に適用されるようになりました。
  * 以前: `_update_session_state()` で `temp:` プレフィックスのキーがスキップされていたため、同一invocation内の後続エージェント（例: `SequentialAgent` の `output_key='temp:my_key'`）が temp ステートを読み取れなかった。
  * 現在: `_apply_temp_state()` が `_trim_temp_delta_state()` の前に呼ばれ、インメモリセッションに反映。永続化は引き続きトリムされる。

### 3.2 DatabaseSessionService の改善

* **スキーマチェックのロック簡素化**: 別途 `_db_schema_lock` を持っていたのが廃止され、`_table_creation_lock` 内で一括管理されるようになりました。
* **行レベルロックの最適化**: `append_event` 時のアプリ/ユーザーステートの行レベルロックが、実際にデルタがある場合のみ取得されるようになりました（`has_app_delta` / `has_user_delta` フラグ）。
* temp ステートの適用が永続化前に行われるよう修正。

### 3.3 Vertex AI Session Service の改善

* `EventCompaction` データが `custom_metadata` 経由で保存・復元されるようになりました（Vertex AI サービスがネイティブでサポートするまでの暫定対応）。
* `usage_metadata` も同様に `custom_metadata` 経由で保存・復元。
* `_from_api_event()` のリファクタリング: `compaction` データを `EventActions` に統合するよう変更。

---

## 4. 認証 (Auth) の拡張

### 4.1 プラガブル AuthProviderRegistry

* **新規ファイル**:
  * `auth/auth_provider_registry.py`: `AuthScheme` の型に基づいて `BaseAuthProvider` インスタンスを登録・取得するレジストリ。
  * `auth/base_auth_provider.py`: カスタム認証プロバイダの抽象基底クラス。`get_auth_credential()` を実装することで、独自の認証フローを組み込める。
* `CredentialManager` が `AuthProviderRegistry` を内部で使用し、ネイティブのトークン取得前にまず登録済みプロバイダをチェックするようになりました。

---

## 5. モデル (Models) の改善

### 5.1 Anthropic LLM

* **PDF ドキュメントサポート**: `DocumentBlockParam` を使用した PDF データの送信に対応（ユーザーターンのみ。アシスタントターンでの PDF は警告を出してスキップ）。
* **ストリーミングサポート**: `_generate_content_streaming()` メソッドが追加され、Anthropic モデルでのストリーミング応答が可能に。`_ToolUseAccumulator` でストリーム中のツール使用ブロックを蓄積。ストリーム中は `partial=True` の `LlmResponse` を yield し、最後に集約済みレスポンスを返す。
* `_update_type_string()` が再帰的にネストされた全ての JSON Schema 構造（`allOf`, `anyOf`, `oneOf`, `$defs`, `prefixItems` 等）を処理するよう大幅改善。
* ツール結果のシリアライズ改善: `dict`/`list` は `json.dumps()` で変換されるようになりました（以前は `str()` で変換）。
* スキーマの `parameters_json_schema` に `copy.deepcopy()` を適用してから変換するよう修正。`model_dump` に `by_alias=True` が追加。

### 5.2 LiteLLM

* **`thought_signature` の保持**: Gemini thinking モデルの `thought_signature` をツールコール間で保持する機能が追加。`_THOUGHT_SIGNATURE_SEPARATOR`（`"__thought__"`）を使用した ID 埋め込み、`provider_specific_fields`、`extra_content.google.thought_signature` の3つのパスをサポート。
* **reasoning 抽出の拡張**: `reasoning_content` に加え `reasoning` フィールドもチェックするようになりました（LM Studio, vLLM 対応）。`.get()` メソッドに統一。
* **`reasoning_tokens`** の抽出: `completion_tokens_details` から推論トークン数を取得する `_extract_reasoning_tokens()` が追加。`UsageMetadataChunk` に `reasoning_tokens` フィールド追加。
* **ストリーミング応答リファクタリング**: `ModelResponse` vs `ModelResponseStream` を `message_field` で判別。`_finalize_tool_call_response()`, `_finalize_text_response()`, `_reset_stream_buffers()` 等の内部関数が整理。`finish_reason="length"` 時に `MAX_TOKENS` エラーを返すように。
* **OpenAI strict スキーマ強制**: `_enforce_strict_openai_schema()` が再帰的に `additionalProperties: false` を追加し、全プロパティを `required` に設定。`$ref` ノードの兄弟キーワードを除去。
* **サポートモデルの大幅拡張**: `azure/`, `azure_ai/`, `bedrock/`, `ollama/`, `together_ai/`, `vertex_ai/`, `mistral/`, `deepseek/`, `fireworks_ai/`, `cohere/`, `databricks/`, `ai21/` 等のプレフィックスが追加。

### 5.3 Apigee LLM

* **`ApiType` enum の追加**: `UNKNOWN`, `CHAT_COMPLETIONS`, `GENAI` の3つのAPI種別をサポート。
* **Chat Completions API サポート**: `api_type=CHAT_COMPLETIONS` 時、Gemini SDK の代わりに `httpx` を使用した直接的な OpenAI 互換リクエスト/レスポンス変換ロジックが追加されました。
* `tenacity` によるリトライ（指数バックオフ）が実装。

### 5.4 Gemini LLM Connection

* **Grounding メタデータの伝播**: `grounding_metadata` が `LlmResponse` に含まれるようになりました（コンテンツが空でもグラウンディングメタデータがある場合はイベントとして出力）。
* 全ての `LlmResponse` yield に `model_version` が含まれるようになりました。

---

## 6. テレメトリ・観測性の強化

### 6.1 新規テレメトリ属性

* `gen_ai.agent.version`: エージェントバージョンのスパン属性が追加。
* `error.type`: ツール実行エラー時のエラータイプがスパンに記録されるようになりました。
* `gen_ai.usage.experimental.reasoning_tokens`: 思考トークン数。
* `gen_ai.usage.experimental.reasoning_tokens_limit`: thinking_budget の制限値。
* `gen_ai.usage.experimental.system_instruction_tokens`: システムインストラクショントークン数。

### 6.2 新規ファイル

* `telemetry/_experimental_semconv.py`: 実験的セマンティック規約のサポート。`gen_ai.client.inference.operation.details` イベントの出力、`gen_ai.tool.definitions` 属性などを管理。
* `telemetry/sqlite_span_exporter.py`: SQLite ベースのスパンエクスポーター。

### 6.3 非同期推論スパン

* `use_inference_span()` が新設され、実験的セマンティック規約のサポートを含むasync版の推論スパン管理が可能に。従来の `use_generate_content_span()` は deprecated に。

---

## 7. エラーハンドリングの改善

### 7.1 新規エラークラス

* **`errors/session_not_found_error.py`**: `SessionNotFoundError(ValueError)` — セッション未検出時の専用例外。後方互換のため `ValueError` を継承。
* **`errors/tool_execution_error.py`**: `ToolExecutionError` — ツール実行エラーの専用例外。OpenTelemetry セマンティクスに準拠した `ToolErrorType` enum（`BAD_REQUEST`, `UNAUTHORIZED`, `NOT_FOUND` 等）を持つ。

### 7.2 Liveモードでのエラーコールバック

* `_execute_function_live()` が大幅にリファクタリングされ、以下が改善されました：
  * `on_tool_error_callback` がLiveモードでも呼び出されるようになりました。
  * ツールが見つからない場合でも `on_tool_error_callback` が呼ばれます。
  * `before_tool_callback` / `after_tool_callback` がLiveモードで正しく動作するようになりました（プラグインコールバック → エージェントコールバックの順序で実行）。

---

## 8. 最適化 (Optimization)

### 8.1 `adk optimize` CLIコマンド

* `cli_tools_click.py` に `optimize` サブコマンドが追加されました。
  * GEPA（Generalized Evolutionary Prompt Architecture）を使用してルートエージェントの instruction を最適化します。
  * `--sampler_config_file_path`（必須）と `--optimizer_config_file_path`（オプション）を受け取ります。

### 8.2 新規ファイル

* `optimization/gepa_root_agent_prompt_optimizer.py`: GEPA ベースのプロンプトオプティマイザ実装。
* `optimization/local_eval_sampler.py`: ローカル評価セットを使用したサンプラー。

---

## 9. A2A (Agent-to-Agent) の大幅刷新

### 9.1 コンバーターの分離・拡張

* **新規ファイル**:
  * `a2a/converters/from_adk_event.py`: ADK Event → A2A イベントへの変換。`convert_event_to_a2a_events()` と `create_error_status_event()` を含む。
  * `a2a/converters/to_adk_event.py`: A2A → ADK Event への変換。型付きコンバーター: `A2AMessageToEventConverter`, `A2ATaskToEventConverter`, `A2AStatusUpdateToEventConverter`, `A2AArtifactUpdateToEventConverter`。
  * `a2a/converters/long_running_functions.py`: `LongRunningFunctions` クラス。複数ターンにまたがる関数呼び出しのタスク状態遷移を管理。
* `part_converter.py`: `thought` メタデータと `thought_signature` のラウンドトリップ（A2A ↔ GenAI Part 間の保持）。ファイル `display_name` の伝播が追加。
* `event_converter.py`: `invocation_context` パラメータが `Optional` に変更。`platform_uuid` / `platform_time` に移行。

### 9.2 Executor の新実装

* `a2a/executor/a2a_agent_executor.py` に `use_legacy` パラメータが追加（デフォルト `True`）。`False` の場合、新実装 `a2a_agent_executor_impl.py` に委譲。
* **`ExecuteInterceptor`**: `before_agent`, `after_event`, `after_agent` の3つのフックポイントを持つインターセプターフレームワーク。`ExecutorContext` がインターセプターに渡される。
* `a2a/executor/config.py`: `A2aAgentExecutorConfig` が分離。インターセプター設定を含む。

### 9.3 リモートエージェント

* **新規ディレクトリ**: `a2a/agent/`
  * `A2aRemoteAgentConfig`: リモート A2A エージェントの構成。
  * `RequestInterceptor`: `before_request` / `after_request` フックでリモート A2A コールをインターセプト。

---

## 10. イベント・UIウィジェット

### 10.1 UiWidget（実験的）

* **新規ファイル**: `events/ui_widget.py`
  * `UiWidget` モデル: `id`, `provider`, `payload` フィールドを持つ。
  * `provider` が `'mcp'` の場合、MCP App の iframe がレンダリングされます。
* `EventActions` に `render_ui_widgets: Optional[list[UiWidget]]` フィールドが追加。

---

## 11. CLI の変更

* `adk run` コマンドで `--memory_service_uri` が実際にサポートされるようになりました（以前は警告が出るだけだった）。`cli.py` 内の `run_input_file()`, `run_interactively()`, `run_cli()` にメモリサービスパラメータが追加。
* `adk optimize` コマンドが新規追加（上記「最適化」セクション参照）。
* `FastAPI` アプリ生成時のエージェントローダーの依存性注入サポート。
* `service_registry.py` に `memory://` スキームが追加され、`--memory_service_uri=memory://` でローカル開発用の `InMemoryMemoryService` が利用可能に。
* `fast_api.py` に A2A プッシュ通知サポート（`InMemoryPushNotificationConfigStore`）が追加。

---

## 12. メモリサービスの拡張

* `BaseMemoryService` に `add_memory()` メソッドが追加されました。
  * `add_events_to_memory()` や `add_session_to_memory()` とは異なり、明示的なメモリエントリを直接書き込むためのインターフェースです。
* Vertex AI Memory Bank: メモリ統合（consolidation）機能、generate/create モードが追加。

---

## 13. 評価 (Evaluation) フレームワーク

### 13.1 User Personas

* **新規ファイル**:
  * `simulation/user_simulator_personas.py`: `UserBehavior`（名前、説明、振る舞い指示、違反ルーブリック）、`UserPersona`（複数の行動を組み合わせたペルソナ）、`UserPersonaRegistry` のデータモデル。
  * `simulation/pre_built_personas.py`: `PreBuiltBehaviors` enum（`ADVANCE_DETAIL_ORIENTED`, `ADVANCE_GOAL_ORIENTED` 等）。デフォルトのペルソナレジストリを提供。
* `ConversationScenario` に `user_persona` フィールドが追加。文字列のペルソナ ID が指定された場合、デフォルトレジストリから解決。

### 13.2 シミュレータのリファクタリング

* プロンプトテンプレートが独立ファイルに分離:
  * `llm_backed_user_simulator_prompts.py`: デフォルトのユーザーシミュレータプロンプト（Jinja2 構文に移行、`{{ stop_signal }}`）。
  * `per_turn_user_simulator_quality_prompts.py`: ペルソナ対応の評価ルーブリック。
* `_VertexAiEvalFacade` が `_SingleTurnVertexAiEvalFacade` にリネーム。
* `evaluation_generator.py`: `user_content` が `Content(parts=[])` で初期化されるよう修正。

---

## 14. アーティファクト (Artifacts)

* `base_artifact_service.py` に `ensure_part()` ユーティリティ関数が追加。`dict` 入力を `types.Part` に正規化（camelCase / snake_case 両対応）。
* `save_artifact()` の `artifact` パラメータの型が `types.Part` から `Union[types.Part, dict[str, Any]]` に拡張。
* `file_artifact_service.py`, `gcs_artifact_service.py`, `in_memory_artifact_service.py` 全てで `ensure_part()` が呼ばれるようになりました。
* `create_time` のデフォルト値が `datetime.now().timestamp()` から `platform_time.get_time()` に変更。

---

## 15. インテグレーション (Integrations) — 新規ディレクトリ

### 15.1 Agent Registry

* **新規**: `integrations/agent_registry/agent_registry.py`
  * Google Cloud Agent Registry（`agentregistry.googleapis.com/v1alpha`）のクライアント。
  * リモート A2A エージェントの検出・一覧取得・インスタンス化。
  * `_ProtocolType` enum: `A2A_AGENT`, `CUSTOM`。
  * レジストリエントリから `RemoteA2aAgent` または `McpToolset` インスタンスを生成。

### 15.2 API Registry

* **新規**: `integrations/api_registry/api_registry.py`
  * Google Cloud API Registry（`cloudapiregistry.googleapis.com`）のクライアント。
  * API Registry に登録された MCP サーバーの `McpToolset` インスタンスを提供。

---

## 16. 依存関係の変更

### 新規追加パッケージ
| パッケージ | バージョン |
|---|---|
| `aiohttp` | 3.13.3 |
| `google-cloud-dataplex` | 2.16.0 |
| `google-cloud-iam` | 2.21.0 |

### 主要パッケージのバージョン変更
| パッケージ | 旧バージョン | 新バージョン |
|---|---|---|
| `google-adk` | 1.25.1 | **1.27.2** |
| `google-genai` | 1.59.0 | **1.68.0** |
| `google-cloud-aiplatform` | 1.133.0 | **1.142.0** |
| `fastapi` | 0.129.0 | 0.135.1 |
| `mcp` | 1.25.0 | 1.26.0 |
| `opentelemetry-api` | 1.37.0 | 1.38.0 |
| `pyarrow` | 22.0.0 | 23.0.1 |
| `starlette` | 0.50.0 | 0.52.1 |
| `grpcio` | 1.76.0 | 1.78.0 |
| `sqlalchemy` | 2.0.45 | 2.0.48 |

---

## 17. 破壊的変更・注意事項

* **Pydantic 下限が 2.7.0 に引き上げ**（v1.26.0）
* `DEFAULT_SKILL_SYSTEM_INSTRUCTION` へのアクセスで deprecation warning が出力されるようになりました（v1.27.0）。後方互換のため再エクスポートは継続。
* LiteLLM ストリーミング応答パースが **LiteLLM 1.81+** 向けにリファクタリングされました。
* `ToolContext` クラスの実装が `agents/context.py` の `Context` クラスに移動。`tool_context.py` は後方互換のための再エクスポートのみ。
* セッション未検出時の例外が `ValueError` → `SessionNotFoundError` に変更。`SessionNotFoundError` は `ValueError` を継承しているため、`except ValueError` で捕捉しているコードは引き続き動作します。
* `use_generate_content_span()` が deprecated になり、`use_inference_span()` に置換。
