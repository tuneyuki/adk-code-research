# Vertex AI Search Tool 戻り値パラメータ & Grounding Metadata 解析

## 概要

`google.adk.agents.Agent` で `VertexAiSearchTool` を使用する場合、検索結果はツールの直接的な出力テキストとしてではなく、Runner (例: `runner.run_async`) が生成する `Event` オブジェクトに埋め込まれる形で返されます。

検索結果が含まれる主要な属性は **`grounding_metadata`** です。

## データ構造

### 1. Agent の戻り値
Agent は `Event` オブジェクトを yield します。
- **クラス**: `google.adk.events.Event` (`LlmResponse` を継承)
- **主要属性**: `event.grounding_metadata`

### 2. Grounding Metadata
`grounding_metadata` には、回答の生成に使用されたソース（根拠）に関する詳細が含まれます。
- **型**: `google.genai.types.GroundingMetadata`

**主要フィールド:**
- `grounding_attributions`: 引用元のリスト（高レベルな情報）。
- `grounding_chunks`: 取得されたコンテンツの詳細なチャンク（断片）。
- `web_search_queries`: 検索に使用されたクエリ。

### 3. Grounding Chunks と URI の有無
`grounding_chunks` は `GroundingChunk` オブジェクトのリストです。ソースがドキュメント (Vertex AI Search) か Web かによって構造が若干異なります。

**重要な点として、`uri` はすべてのケースにおいて Optional (任意項目) です。**

#### A. ドキュメント検索 (`retrieved_context`)
Datastore 内の非構造化データ（PDFなど）を検索する場合。

- **クラス**: `google.genai.types.GroundingChunkRetrievedContext`
- **フィールド**:
  - `uri`: `Optional[str]` (ソースURI, 例: gs://..., https://...) - **None の可能性があります**
  - `title`: `Optional[str]`
  - `text`: `Optional[str]` (本文の抜粋)
  - `document_name`: `Optional[str]`

#### B. Web 検索 (`web`)
一般の Web を検索する場合。

- **クラス**: `google.genai.types.GroundingChunkWeb`
- **フィールド**:
  - `uri`: `Optional[str]` - **None の可能性があります**
  - `title`: `Optional[str]`

## 実装上の推奨事項

`uri` の存在は保証されない（Datastore の設定やデータタイプに依存する）ため、コード側では `None` の場合を考慮して適切に処理する必要があります。

```python
# 安全なアクセス例
for chunk in event.grounding_metadata.grounding_chunks:
    # retrieved_context の確認 (Vertex AI Search の典型パターン)
    if chunk.retrieved_context:
        uri = chunk.retrieved_context.uri
        title = chunk.retrieved_context.title or "タイトルなし"
        print(f"ソース: {title} ({uri if uri else 'URIなし'})")
        
    # web の確認 (検索タイプが混合または Web 検索の場合)
    elif chunk.web:
        uri = chunk.web.uri
        title = chunk.web.title or "タイトルなし"
        print(f"Webソース: {title} ({uri if uri else 'URIなし'})")
```
