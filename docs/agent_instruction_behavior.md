# AgentのInstructionの動的変更について

このドキュメントでは、実行中のAgentインスタンスの `instruction` プロパティを変更した場合の挙動について解説します。

## 結論

**可能であり、即座に反映されます。**

`Agent` インスタンスの `instruction` 属性をPythonコード上で書き換えると、その変更は**同一インスタンスを使用している全ての進行中および新規のセッションの次のターン**から即座に有効になります。

## 挙動の検証結果

### シナリオ
1.  Instruction="Your name is OLD_NAME" で Agent を初期化。
2.  Sessionを開始し、1ターン目の会話を行う。 -> 回答: "I am OLD_NAME"
3.  Pythonコードで `agent.instruction = "Your name is NEW_NAME"` と書き換え。
4.  同じSessionIDで2ターン目の会話を行う。

### 結果
2ターン目の回答は **"I am NEW_NAME"** となり、変更されたInstructionに従った振る舞いとなります。

## 仕組み

`Runner` クラスは `Agent` インスタンスを保持（参照）しています。
`run()` または `run_async()` が呼び出されるたびに、その時点での `agent.instruction` の値を参照してプロンプトを構築するため、インスタンス変数の変更が即座に反映されます。

## 注意点

*   **グローバルな影響**: `Agent` インスタンスは通常、アプリケーション内でシングルトンとして扱われます（`main.py` 内で定義され、全リクエストで共有される）。したがって、`agent.instruction` を書き換えると、**そのAgentを使用している他の全ユーザー/セッション**にも影響が及びます。
*   **セッションごとの個別Instruction**: セッション (ユーザー) ごとに異なるInstructionを与えたい場合は、Agent自体のInstructionを書き換えるのではなく、`run()` メソッドの引数などでコンテキストごとの追加プロンプトを渡すか、セッションごとに個別のAgentインスタンスを生成する（ただしコストが高い）等の設計が必要です。
    *   *補足*: 現在のADKの `Runner.run` メソッドには、ターンごとのInstructionオーバーライド用パラメータは直接的には存在しないため、動的に変更したい場合は注意が必要です。

## 実装例: クライアントからのパラメータで変更する

Web APIサーバー（FastAPIなど）として実装する場合の、具体的なコード例です。
リクエストボディに含まれる `system_mode` というパラメータに応じて、動的にAgentのInstructionを書き換えてから実行します。

```python
from fastapi import FastAPI
from pydantic import BaseModel
from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types

app = FastAPI()

# 1. Agent定義 (グローバルスコープ)
# デフォルトのInstruction
DEFAULT_INSTRUCTION = "You are a helpful assistant."
agent = Agent(
    name="my_agent",
    model="gemini-2.0-flash",
    instruction=DEFAULT_INSTRUCTION
)

session_service = InMemorySessionService()
runner = Runner(agent=agent, app_name="my_app", session_service=session_service)

# リクエストモデル
class ChatRequest(BaseModel):
    user_id: str
    session_id: str
    message: str
    system_mode: str = "normal"  # クライアントからの制御用パラメータ

@app.post("/chat")
async def chat_endpoint(req: ChatRequest):
    # 2. リクエストごとのInstruction動的変更
    # 注意: Agentはシングルトンなので、この変更は他リクエストと競合する可能性があります
    # (同時アクセス・並列処理がある場合は、Lockを使用するか、リクエスト毎にAgentを複製する必要があります)
    
    if req.system_mode == "pirate":
        agent.instruction = "You are a pirate captain. Answer everything with 'Arrr!' and pirate slang."
    elif req.system_mode == "formal":
        agent.instruction = "You are a formal butler. Be extremely polite and concise."
    else:
        # デフォルトに戻す (これを忘れると前の設定が残ります)
        agent.instruction = DEFAULT_INSTRUCTION

    # 3. 実行
    # 変更されたInstructionがこのターンの実行に使われます
    content = types.Content(role='user', parts=[types.Part(text=req.message)])
    
    responses = []
    async for event in runner.run_async(
        user_id=req.user_id, 
        session_id=req.session_id, 
        new_message=content
    ):
        if event.is_final_response():
             responses.append(event.content.parts[0].text)

    return {"response": "".join(responses)}
```

### 並行処理時の注意（重要）

ユーザーが異なり `SessionID` が異なっていても、**`Agent` インスタンスそのものがサーバー内で1つ（シングルトン）として共有されている場合**は競合が起きます。

#### なぜ起きるのか？

*   **Session**: 会話の「履歴」や「コンテキスト」です。これは `SessionID` ごとに独立しています。
*   **Agent**: 「性格」や「設定」です。`main.py` などで `agent = Agent(...)` と定義した**その単一のオブジェクト**を、すべてのセッションが参照しています。

もし User A のリクエスト処理で `agent.instruction` を書き換えると、その瞬間にメモリ上の Agent オブジェクト自体が変化します。
その直後に（User Aの処理が終わる前に）User B のリクエストが来て `agent.instruction` を別の値に書き換えると、User A が次に Agent を参照したときには、User B 用の設定に変わってしまっていることになります。

#### 安全な実装（コピーの作成）

この問題を回避するためには、リクエストごとに Agent のコピー（複製）を作成し、その複製に対して変更を加えます。

```python
    import copy
    
    # 1. テンプレートとなる元のAgent (グローバル変数)
    # global_agent = ...
    
    # 2. リクエスト専用のコピーを作成 (浅いコピーで通常は十分です)
    # これにより、current_agentへの変更は global_agent に影響しません
    current_agent = copy.copy(agent) 
    
    # 3. コピーに対してInstructionを変更
    current_agent.instruction = new_instruction_text
    
    # 4. コピーしたAgentを使ってRunnerを作成・実行
    temp_runner = Runner(agent=current_agent, ...)
    
    await temp_runner.run_async(...)
```

## Cloud Run / 本番環境での推奨パターン: Instance per Request

「シングルトン（グローバル変数への依存）をやめる」ことは可能ですし、Cloud Run のようなステートレスなコンテナ環境では、より安全な設計となります。

具体的には、グローバルスコープで `agent` 変数を作らず、**「リクエストを受け取った関数の中で Agent を `new`（インスタンス化）する」** デザインにします。

### コード例

```python
from fastapi import FastAPI
from pydantic import BaseModel
from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types

app = FastAPI()

# 注: ここに `agent = ...` を書かない！

@app.post("/chat-stateless")
async def chat_stateless(req: ChatRequest):
    # 1. リクエストの中で Agent を生成する (Instance per Request)
    # これにより、この `agent` インスタンスはこの関数スコープ (つまりこのリクエスト) 内でのみ存在します。
    # 競合の心配はゼロになります。
    
    # 必要ならリクエストパラメータに基づいて設定を変えて初期化
    instruction_text = "You are a helpful assistant."
    if req.system_mode == "pirate":
        instruction_text = "You are a pirate captain."
        
    # ここで生成 (軽量です)
    agent = Agent(
        name="my_dynamic_agent",
        model="gemini-2.0-flash",
        instruction=instruction_text
    )

    # 2. Runnerもここで生成
    # SessionServiceは永続化層(Firestore等)を使う場合はグローバルで一つ持っておくのが普通ですが、
    # InMemoryの場合はここで作るか、外部DBへ接続します。
    session_service = InMemorySessionService() 
    runner = Runner(agent=agent, app_name="my_app", session_service=session_service)

    # 3. 実行 (通常通り)
    content = types.Content(role='user', parts=[types.Part(text=req.message)])
    # ... runner.run_async(...)
```

### メリット・デメリット

*   **メリット**:
    *   **完全なスレッドセーフ**: 他のリクエストの影響を絶対に受けません。
    *   **柔軟性**: ユーザーごと、リクエストごとに全く異なる設定のAgentを生成できます。
*   **デメリット**:
    *   **オーバーヘッド**: リクエストごとにPythonオブジェクトの生成コストがかかります。ただし、ADKの `Agent` クラス自体は設定保持用の軽量なオブジェクトであり、巨大なモデルウェイトをロードするわけではない（APIクライアントとして振る舞う）ため、これを都度生成するコストは**無視できるほど小さい**です。
    *   **コネクション**: 内部でAPIクライアント (`google.genai.Client`) を生成する場合、TCPコネクションの確立などが頻発する可能性があります。パフォーマンスを極限まで気にする場合は、`Client` オブジェクトだけはグローバル (シングルトン) にして、Agent生成時に渡すなどの工夫が有効です。

### 動作検証結果

実機検証の結果、この「Instance per Request」パターンで問題なく動作することを確認済みです。
1.  リクエスト1: "Polite Butler" Agent生成 -> 会話 ("I am UserA")
2.  リクエスト2: "Pirate" Agent生成 -> 会話 ("What is my name?") -> 回答 "UserA!" (履歴も保持される)

これらがエラーなく動作し、かつ設定が動的に切り替わることが確認されています。



