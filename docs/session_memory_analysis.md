# run_sse実行中にクライアントが切断された場合のSessionMemoryの状態分析

## 結論

クライアントが `run_sse` (Server-Sent Events) でのレスポンス受信中に切断した場合、**生成中だったレスポンス（テキスト）は `SessionMemory` に保存されません。**

セッションメモリには、**切断前に完了していたイベントのみ**が保存された状態となります。

---

## ソースコードレベルでの確認

### 1. `run_sse` の処理フロー

`google.adk.cli.adk_web_server.py` の `run_agent_sse` エンドポイントがリクエストを処理します。

```python
# google/adk/cli/adk_web_server.py

@app.post("/run_sse")
async def run_agent_sse(req: RunAgentRequest) -> StreamingResponse:
  # ...
  runner = await self.get_runner_async(req.app_name)
  async with Aclosing(
      runner.run_async(
          # ...
          run_config=RunConfig(streaming_mode=StreamingMode.SSE),
          # ...
      )
  ) as agen:
    async for event in agen:
      # ...
      yield f"data: {sse_event}\n\n"
```

レスポンスは `StreamingResponse` を介してクライアントにストリーミングされます。
`Runner.run_async` からイベント (`Event`) が `yield` されるたびに、SSEイベントとしてクライアントに送信しようとします。

### 2. イベントの保存 (`append_event`)

`Runner.run_async` は内部で `_run_with_trace` -> `_exec_with_plugin` を呼び出します。
`google.adk.runners.py` 内で、イベントがエージェントから生成されると、**`yield` する前に** `session_service.append_event` が呼び出されます。

```python
# google/adk/runners.py

async def _exec_with_plugin(...):
  # ...
  async with Aclosing(execute_fn(invocation_context)) as agen:
    async for event in agen:
      # ...
      # 保存処理 (非Live呼び出しの場合)
      await self.session_service.append_event(
          session=session, event=event
      )
      
      # 変更後のイベントをyield (これが adk_web_server に渡る)
      yield modified_event
```

一見すると、`append_event` が呼ばれているため保存されるように見えますが、**`InMemorySessionService` の実装に重要なガード条件**があります。

### 3. Partial (途中経過) イベントの無視

`InMemorySessionService` の `append_event` 実装を確認すると、**`event.partial` が True の場合、保存処理をスキップ**しています。

```python
# google/adk/sessions/in_memory_session_service.py

@override
async def append_event(self, session: Session, event: Event) -> Event:
  if event.partial:
    # 部分的なイベントは保存せずに即リターン
    return event

  # ... (以下、保存処理)
```

### 4. ストリーミング中のイベントの状態

`LlmAgent` および `BaseLlmFlow` (`google.adk.flows.llm_flows.base_llm_flow.py`) では、`StreamingMode.SSE` が有効な場合、生成中のテキストチャンクを `partial=True` のイベントとして送出します。

- **ストリーミング中**: エージェントは `partial=True` のイベントを次々と生成します。
- **保存処理**: `InMemorySessionService.append_event` はこれらを**無視**します。
- **クライアント送信**: `adk_web_server` はこれらをクライアントに送信します。

### 5. クライアント切断時の挙動

クライアントが切断すると、`adk_web_server` 側のジェネレータが停止・キャンセルされます。これにより `Runner` 側の処理も中断されます。

- **正常終了時**: 生成が完了すると、最後に `partial=False` の完全なイベント（または `turn_complete` 等）が生成され、これが `append_event` で保存されます。
- **切断時**: 処理が中断されるため、最後の「完全なイベント (`partial=False`)」が生成される行まで到達しません。

### まとめ: SessionMemoryの状態

したがって、切断時点での `SessionMemory` の状態は以下のようになります。

1. **ユーザーの入力メッセージ**: 正常に保存されています（処理開始前に保存されるため）。
2. **直前のTool呼び出し**: ストリーミング回答の前に Tool 呼び出しなどが完了していれば、その Tool Request / Tool Response イベントは保存されています（これらは通常 `partial=False` です）。
3. **生成中の回答**: **保存されていません。**

システムの観点からは、「回答の生成を試みたが、何も出力されずに終わった（あるいは回答が存在しない）」状態に見えます。
