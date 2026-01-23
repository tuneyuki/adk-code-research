# SSE Yield途中でのクライアント切断とBufferの扱い

## 結論

クライアントが切断されると、**Buffer（Aggregator内に蓄積されたテキスト）は破棄され、消失します。**

ソースコード上、Bufferの内容を `Event` として取り出して保存する処理（`aggregator.close()`）は、ストリーミングループの **後** に記述されています。
切断によってループが強制中断（キャンセル）されると、この終了処理に到達することなく関数が終了するため、Buffer内のデータはメモリから解放されて消えます。

---

## ソースコードレベルでの確認

### 1. バッファリングとループ処理 (`google_llm.py`)

`generate_content_async` メソッド内の処理を確認します。

```python
# google/adk/models/google_llm.py

# Aggregatorのインスタンス化（ここでBufferがメモリ上に作られる）
aggregator = StreamingResponseAggregator()

async with Aclosing(responses) as agen:
  async for response in agen:
    # ...
    async with Aclosing(
        aggregator.process_response(response)
    ) as aggregator_gen:
      async for llm_response in aggregator_gen:
        # Partialな結果をYield（これはSessionには保存されない）
        yield llm_response

# 【重要】ここが到達不能コードになる
if (close_result := aggregator.close()) is not None:
  # ...
  yield close_result  # ここでFinalなEventが出るはずだった
```

### 2. 切断時の動作メカニズム

1. **Client Disconnect**: クライアントが切断すると、Webサーバー（Uvicorn/FastAPI）は接続の終了を検知します。
2. **Cancellation**: Pythonの `asyncio` タスクとして実行されている `run_agent_sse` に対して `CancelledError` が送出されるか、あるいは下流の `yield` が停止することで、上流のジェネレータ（上記の `async for` ループ）も停止します。
3. **Cleanup**: `async with Aclosing(...)` によって、ジェネレータの `aclose()` が呼ばれ、リソースはクリーンアップされます。
4. **Skipped Logic**: ループを抜けた **後** にある `aggregator.close()` の呼び出し行には、制御が移りません（例外による中断、またはジェネレータの停止のため）。

### 3. 結果

`aggregator` インスタンスはローカル変数であるため、関数スコープを抜けた時点で参照がなくなり、ガベージコレクションの対象となります。
`StreamingResponseAggregator` クラス自体にはデストラクタ（`__del__`）やコンテキストマネージャ（`__aexit__`）による「破棄時の自動保存」ロジックは実装されていません（`streaming_utils.py` 確認済み）。

したがって、**蓄積されていた `aggregator` 内部のテキストデータは、どこにも保存されることなく消失します。**
