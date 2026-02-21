# ADK × VertexAiSearchTool: Grounding Metadata の取得と データストア構築ガイド

## 概要

ADK の `VertexAiSearchTool` をエージェントのツールとして使用したとき、検索結果は通常のツール戻り値としてではなく、**`event.grounding_metadata`** に格納されて返却される。

本ドキュメントでは以下を解説する。

1. Grounding Metadata がどのような構造で返却されるか
2. `retrieved_context.uri` / `title` / `text` を正しく取得するためのデータストア構築方法
3. Agent・Runner の実装コードと Grounding Metadata の抽出コード
4. 実測した検証結果

---

## 1. VertexAiSearchTool を使った Agent の基本実装

```python
import os
import asyncio
import uuid
from google.adk.tools.vertex_ai_search_tool import VertexAiSearchTool
from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions.in_memory_session_service import InMemorySessionService
from google.genai import types

# Vertex AI を使うための環境変数設定
os.environ["GOOGLE_GENAI_USE_VERTEXAI"] = "1"
os.environ["GOOGLE_CLOUD_PROJECT"] = "your-project-id"
os.environ["GOOGLE_CLOUD_LOCATION"] = "us-central1"

# データストアの完全リソース名
full_ds_id = (
    "projects/your-project-id/locations/global"
    "/collections/default_collection"
    "/dataStores/your-datastore-id"
)

tool = VertexAiSearchTool(data_store_id=full_ds_id)

agent = Agent(
    name="faq_agent",
    instruction="ユーザーの質問に対してツールを使って検索し、回答してください。",
    model="gemini-2.5-flash",   # Vertex AI 上に存在するモデルを指定
    tools=[tool]
)

session_service = InMemorySessionService()
runner = Runner(
    app_name="faq_app",
    agent=agent,
    session_service=session_service,
    auto_create_session=True
)
```

> **注意**: `gemini-2.5-pro-preview` や `gemini-3.x` など、直接 Gemini API には存在するが
> Vertex AI エンドポイントに存在しないモデルを指定すると 404 エラーになる。
> `GOOGLE_GENAI_USE_VERTEXAI=1` を設定している場合は必ず Vertex AI 対応モデルを使用すること。

---

## 2. Grounding Metadata の取り出し方

`runner.run_async()` が yield する `Event` オブジェクトから取得する。

```python
async def query(runner, query_text: str):
    session_id = str(uuid.uuid4())   # クエリごとに新しいセッションを作成
    new_msg = types.Content(
        role="user",
        parts=[types.Part.from_text(text=query_text)]
    )

    full_response = ""
    grounding_metadata = None

    async for event in runner.run_async(
        user_id="user", session_id=session_id, new_message=new_msg
    ):
        # Agent の回答テキスト
        if event.content and event.content.parts:
            for p in event.content.parts:
                if p.text:
                    full_response += p.text

        # Grounding Metadata（検索根拠）
        if event.grounding_metadata:
            grounding_metadata = event.grounding_metadata

    return full_response, grounding_metadata
```

> **ポイント: セッションはクエリごとに新規作成すること**
> 同一セッションで複数クエリを続けると、モデルが会話コンテキストから回答を生成し
> ツールを呼ばなくなる（chunks=0）ケースが増える。テストや検証では 1 クエリ = 1 セッションにする。

---

## 3. Grounding Metadata の構造

```
event.grounding_metadata          # google.genai.types.GroundingMetadata
  └─ .grounding_chunks            # List[GroundingChunk]
       └─ .retrieved_context      # GroundingChunkRetrievedContext (Vertex AI Search の場合)
            ├─ .uri               # str | None  ← インポート時トップレベル uri フィールド
            ├─ .title             # str | None  ← インポート時トップレベル title フィールド
            ├─ .text              # str | None  ← ドキュメント全体のシリアライズ（後述）
            └─ .document_name     # str | None  ← 内部リソースパス（変更不可）
```

### `retrieved_context.text` の実際の内容

`text` は特定フィールドのマッピングではなく、**ドキュメント全体を YAML 風にシリアライズしたもの**が返却される。
structData の中に `Answer` と `copyright` だけを格納した場合の実際の返却値：

```
structData:
  Answer: "検査に当たっては、国が行う工事や地方公共団体が国の補助金で行う工事などについて..."
  copyright: 会計検査院
title: 公共工事について会計検査院はどのように検査をしているのですか。
uri: https://www.jbaudit.go.jp/general/faq.html
```

- `Answer` の内容は `text` 内の `structData.Answer` から取得できる（アプリ側でパース）
- `copyright` も `text` 内の `structData.copyright` に含まれる
- `retrieved_context` には専用の copyright フィールドは存在しない（`uri`/`title`/`text`/`document_name` の4フィールドのみ）

### Grounding Metadata の解析コード

```python
chunks_info = []
if grounding_metadata and grounding_metadata.grounding_chunks:
    for chunk in grounding_metadata.grounding_chunks:
        if chunk.retrieved_context:
            rc = chunk.retrieved_context
            chunks_info.append({
                "uri":   rc.uri,
                "title": rc.title,
                "text":  rc.text,
            })
        elif chunk.web:
            # Web 検索の場合
            chunks_info.append({
                "uri":   chunk.web.uri,
                "title": chunk.web.title,
            })
```

---

## 4. `uri` / `title` を返すためのデータストア構築方法

### 問題の背景

`retrieved_context.uri` と `retrieved_context.title` はデフォルトでは `None` になる。
Discovery Engine はドキュメントの **トップレベルの `uri` フィールド** と **`title` フィールド** を
それぞれ `retrieved_context.uri` / `retrieved_context.title` にマッピングする。

スキーマの `fieldConfigs` で `keyPropertyType: "uri"` を設定する API 的な手段もあるが、
`NO_CONTENT` 型の構造化データストアでは **インポートデータのトップレベルに `uri` と `title` を含める方法が確実**。

---

### 4-1. データストアの種類

本ガイドでは **NO_CONTENT（コンテンツなし・構造化データのみ）** のデータストアを対象とする。

```python
from google.cloud import discoveryengine

data_store = discoveryengine.DataStore(
    display_name="FAQ Store",
    industry_vertical=discoveryengine.IndustryVertical.GENERIC,
    content_config=discoveryengine.DataStore.ContentConfig.NO_CONTENT,
)
```

---

### 4-2. インポートデータの形式（重要）

各ドキュメントを JSONL 形式でインポートする際、以下の構造にする。

```json
{
  "_id":       "ユニークなID（必須）",
  "uri":       "https://example.com/page.html",
  "title":     "Q: ○○についてどう思いますか？",
  "structData": {
    "Question":  "○○についてどう思いますか？",
    "Answer":    "○○については〜です。",
    "url":       "https://example.com/page.html",
    "copyright": "○○省"
  }
}
```

| フィールド | 説明 | Grounding Metadata との対応 |
|-----------|------|---------------------------|
| `uri` (トップレベル) | ソース URL | → `retrieved_context.uri` |
| `title` (トップレベル) | ドキュメントの見出し（Question） | → `retrieved_context.title` |
| `structData.Answer` | 回答本文 | → `retrieved_context.text` 内に含まれる |
| `structData.copyright` | 著作権者（フィルタ用） | → `retrieved_context.text` 内に含まれる |

**重要**: `structData` には `retrieved_context.text` に出したいフィールドのみを格納する。
元データの `url` や `Question` はトップレベルの `uri`/`title` で表現できるため、`structData` に重複して入れない。

#### フォーマット変換コード例

```python
import json, uuid

def format_jsonl_for_import(input_file: str, output_file: str):
    """
    元データ JSONL → Discovery Engine インポート用 JSONL に変換。

    フィールドの割り当て：
      uri   <- url       → retrieved_context.uri
      title <- Question  → retrieved_context.title
      structData.Answer    <- Answer    → retrieved_context.text 内
      structData.copyright <- copyright → retrieved_context.text 内 / フィルタ用
    """
    with open(input_file, "r", encoding="utf-8") as infile, \
         open(output_file, "w", encoding="utf-8") as outfile:
        for line in infile:
            data = json.loads(line)
            formatted = {
                "_id":   uuid.uuid4().hex,
                "uri":   data.get("url", ""),       # → retrieved_context.uri
                "title": data.get("Question", ""),  # → retrieved_context.title
                "structData": {
                    "Answer":    data.get("Answer", ""),    # → text 内
                    "copyright": data.get("copyright", ""), # → text 内 / filter 用
                }
            }
            outfile.write(json.dumps(formatted, ensure_ascii=False) + "\n")
```

---

### 4-3. インポート方法

```python
from google.cloud import discoveryengine

def import_documents(project_id, data_store_id, gcs_uri, full=False):
    client = discoveryengine.DocumentServiceClient()
    parent = client.branch_path(project_id, "global", data_store_id, "default_branch")

    mode = (
        discoveryengine.ImportDocumentsRequest.ReconciliationMode.FULL
        if full else
        discoveryengine.ImportDocumentsRequest.ReconciliationMode.INCREMENTAL
    )

    request = discoveryengine.ImportDocumentsRequest(
        parent=parent,
        gcs_source=discoveryengine.GcsSource(
            input_uris=[gcs_uri],
            data_schema="custom"   # 構造化データ（structData）を使う場合
        ),
        reconciliation_mode=mode
    )

    operation = client.import_documents(request=request)
    response = operation.result()
    print(f"Import completed: {response}")
```

> **`FULL` モードについて**
> `FULL` を指定するとデータストア内の既存ドキュメントをすべて削除し、新規インポートで置き換える。
> フォーマット変更後の再インポートなど「全件入れ替え」が必要な場合に使用する。

---

### 4-4. copyright フィールドのフィルタリング設定

`copyright` などのフィールドを `VertexAiSearchTool` の `filter` パラメータで絞り込む場合、
スキーマで `indexable: true` を設定する必要がある。

#### スキーマ例（jsonSchema 文字列として指定）

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "structData": {
      "type": "object",
      "properties": {
        "url":       { "type": "string", "indexable": true, "searchable": true, "retrievable": true },
        "Question":  { "type": "string", "indexable": true, "searchable": true, "retrievable": true },
        "Answer":    { "type": "string", "indexable": true, "searchable": true, "retrievable": true },
        "copyright": { "type": "string", "indexable": true, "searchable": true, "retrievable": true }
      }
    }
  }
}
```

`indexable: true` が設定されたフィールドは、検索フィルタ式で使用可能になる。

#### フィルタを使った VertexAiSearchTool の呼び出し

```python
tool = VertexAiSearchTool(
    data_store_id=full_ds_id,
    filter='structData.copyright: ANY("会計検査院")'
)
```

**重要**: フィルタ式のフィールドパスは **`structData.copyright`** と `structData.` プレフィックスが必要。
`copyright: ANY(...)` と書くと `Unsupported field` エラーになる。

また、`ANY` 演算子（`:`）はサポートされているが、`=` 演算子はサポートされない：
- ✅ `structData.copyright: ANY("会計検査院")`
- ❌ `copyright: ANY("会計検査院")`  ← structData. プレフィックスが必要
- ❌ `structData.copyright = "会計検査院"` ← = 演算子は非サポート

#### フィルタ効果の実測値（同一クエリ・3パターン比較）

| パターン | chunks数 | 返却された省庁 |
|---|---|---|
| フィルタなし | 82 | meti.go.jp, mhlw.go.jp など複数 |
| `structData.copyright: ANY("会計検査院")` | 12 | jbaudit.go.jp のみ |
| `structData.copyright: ANY("経済産業省")` | 42 | meti.go.jp のみ |

---

## 5. 実測結果

### テスト設定

- データ: 政府系FAQ データ 22,794 件（Question / Answer / url / copyright）
- データストア: NO_CONTENT 型（構造化データのみ）
- モデル: `gemini-2.5-flash`（Vertex AI エンドポイント）
- テストクエリ数: 11件（data.jsonl の 1行目・1001行目・…・10001行目）
- 判定基準: `retrieved_context.uri` に期待する URL が含まれるか

### 結果サマリー

| 条件 | ヒット率 |
|------|---------|
| 旧フォーマット（`uri`/`title` なし）| **0/11（0%）** |
| 新フォーマット（`uri`/`title` あり）・同一セッション | **4/11（36%）** |
| 新フォーマット（`uri`/`title` あり）・クエリごとに新セッション | **10/11（91%）** |

### 実際に返却された Grounding Metadata のサンプル

```
クエリ: 「公共工事について会計検査院はどのように検査をしているのですか。ということですが、これについて詳しく教えてもらえますか？」

chunk[0]:
  uri   = https://www.jbaudit.go.jp/general/faq.html
  title = 公共工事について会計検査院はどのように検査をしているのですか。

chunk[1]:
  uri   = https://www.jbaudit.go.jp/general/faq.html
  title = 公共工事について会計検査院はどのような指摘をしていますか。

chunk[2]:
  uri   = https://www.jbaudit.go.jp/general/faq.html
  title = 工事の検査を行うには、専門的な知識が必要だと思いますが、工事検査のために技術系の職員を多く採用しているのですか。

（以降 26 chunk）
```

- `uri`: インポート時トップレベルに設定した `uri`（= 元データの `url` フィールド）が正しく返却
- `title`: インポート時トップレベルに設定した `title`（= 元データの `Question` フィールド）が正しく返却
- 1件のクエリに対して複数ドキュメントの複数チャンクが返却される（上記例では 26 chunk）
- 同一 URL から複数チャンク（複数の Question）が返却されることがある

### ヒットしなかった 1 件について

| Index | Question | 結果 |
|-------|----------|------|
| 4001 | 「申請内容を修正したいです。」 | chunks=0（ツール未呼び出し） |

クエリが短く汎用的なため、モデルが自分の知識で回答できると判断しツールを呼ばなかった。
データストア・グラウンディングの問題ではない。

---

## 6. まとめ・チェックリスト

### データストア構築

- [ ] データを JSONL 形式に変換し、トップレベルに `_id`・`uri`・`title` を追加する
- [ ] `uri` には元データの URL フィールドを設定する（→ `retrieved_context.uri` に反映）
- [ ] `title` には見出しとなるフィールド（Q&A なら Question）を設定する（→ `retrieved_context.title` に反映）
- [ ] フィルタ対象フィールドはスキーマで `indexable: true` を設定する
- [ ] インポートは `data_schema="custom"`、全件置換は `ReconciliationMode.FULL` を使う

### Agent 実装

- [ ] `GOOGLE_GENAI_USE_VERTEXAI=1` + `GOOGLE_CLOUD_PROJECT` + `GOOGLE_CLOUD_LOCATION` を設定する
- [ ] モデルは Vertex AI 上に存在するものを使う（`gemini-2.5-flash` など）
- [ ] `VertexAiSearchTool` の `data_store_id` は完全リソース名（`projects/.../dataStores/...`）で指定する
- [ ] テスト・検証時はクエリごとに新しい `session_id` を発行する

### Grounding Metadata の取り出し

- [ ] `event.grounding_metadata.grounding_chunks` をイテレートする
- [ ] Vertex AI Search の場合は `chunk.retrieved_context` を参照する
- [ ] `uri`・`title`・`text` はすべて `Optional[str]`（`None` チェックが必要）
