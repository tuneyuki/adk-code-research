# Webサーバーにおけるクライアント切断検知メカニズム (Deep Dive)

## 概要

クライアント(ブラウザ等)からの切断は、OSレベルのTCP接続終了から始まり、ASGIサーバー(Uvicorn)を経てWebフレームワーク(Starlette/FastAPI)に伝わり、最終的にアプリケーションタスクのキャンセル(`CancelledError`)として処理されます。

この一連の流れにより、ADKのストリーミング生成処理(`run_agent_sse`)は強制的に中断され、結果としてバッファリングされていたデータはメモリ上から消失します。

## シーケンス詳細 (Mermaid)

```mermaid
sequenceDiagram
    participant Client
    participant OS
    participant Uvicorn as Uvicorn (ASGI Server)
    participant Starlette as Starlette (Framework)
    participant ADK as ADK (Application)
    participant Aggregator as Buffer (Aggregator)

    Note over Client, ADK: 正常なストリーミング中
    ADK->>Aggregator: Buffer Data
    ADK->>Starlette: yield Chunk
    Starlette->>Uvicorn: send(body)
    Uvicorn->>Client: TCP Send

    Note over Client, OS: ユーザーがブラウザを閉じる (切断)
    Client-xOS: TCP FIN / RST
    OS->>Uvicorn: Connection Lost (Callback)
    
    rect rgb(255, 240, 240)
        Note over Uvicorn: Uvicorn Protocol Layer
        Uvicorn->>Uvicorn: connection_lost()
        Uvicorn->>Uvicorn: state.disconnected = True
        Uvicorn-->>Starlette: Queue "http.disconnect" event
    end

    rect rgb(240, 240, 255)
        Note over Starlette: Starlette StreamingResponse
        Starlette->>Starlette: listen_for_disconnect() 
        Note right of Starlette: 常駐タスクがreceive()で<br/>切断メッセージを検知
        Starlette->>Starlette: Cancel Task Group
    end

    rect rgb(255, 255, 240)
        Note over ADK: Application Handling
        Starlette-xADK: Raise asyncio.CancelledError
        Note right of ADK: ジェネレータの実行が<br/>現在行で強制中断
        ADK->>ADK: async with Aclosing logic (Cleanup)
        Note right of ADK: ループ後の aggregator.close() は<br/>実行されずにスキップ
    end

    Note over Aggregator: 参照が失われ、GCで破棄 (データ消失)
```

## ソースコードレベルの裏付け

### 1. Uvicorn: 切断イベントの発行
`uvicorn/protocols/http/httptools_impl.py` (または `h11_impl.py`) において、OSからの切断通知を受け取ると、ASGIの `receive` チャンネルに `http.disconnect` メッセージをキューイングします。

```python
# Uvicorn (Conceptual)
def connection_lost(self, exc):
    self.cycle.disconnected = True
    self.cycle.message_event.set() # receive()待ちを解除

async def receive(self):
    if self.disconnected:
        return {"type": "http.disconnect"}
```

### 2. Starlette: 切断の監視とキャンセル
`starlette/responses.py` の `StreamingResponse` クラスは、レスポンスの送信(`stream_response`)と並行して、切断信号の監視(`listen_for_disconnect`)を行います。

```python
# Starlette (starlette/responses.py)
async def listen_for_disconnect(self, receive: Receive) -> None:
    while True:
        message = await receive()
        if message["type"] == "http.disconnect":
            break # 切断検知

async def __call__(self, scope, receive, send):
    async with anyio.create_task_group() as task_group:
        # 送信タスクと監視タスクを並行実行
        task_group.start_soon(self.stream_response, send)
        await self.listen_for_disconnect(receive)
        
        # 監視タスクが終了(切断検知)すると、グループ全体をキャンセル
        task_group.cancel_scope.cancel() 
```

### 3. ADK: 中断とバッファ消失
Starletteによるタスクキャンセルは、Pythonの `asyncio.CancelledError` としてADKのコード内で発生します。これにより `run_agent_sse` 内のジェネレータは即座に停止し、関数末尾にある保存処理(`aggregator.close()`)に到達することなく終了します。

```python
# ADK (google/adk/models/google_llm.py)
aggregator = StreamingResponseAggregator()
try:
    # ... ストリーミングループ ...
    yield partial_response # <--- ここで CancelledError 発生
except GeneratorExit:
    pass # クリーンアップのみ実行

# ↓ この行には永遠に到達しない
if (close_result := aggregator.close()) is not None:
   yield close_result
```

### 結論
- ユーザーの推測通り、**Uvicornが切断を検知**します。
- その後、**Starletteがそれを拾ってタスクをキャンセル**します。
- その結果、ADKの処理が中断され、**バッファは保存されずに消滅**します。
