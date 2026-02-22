# BigQuery → Vertex AI Search データストア統合ガイド

> 調査日: 2026-02-21

---

## 概要

**Vertex AI Search**（AI Applications の一部）は、BigQuery に蓄積された構造化データを直接インポートし、セマンティック検索・Gemini グラウンディング（RAG）に活用できる仕組みです。

既存の BigQuery 資産を追加の ETL なしに、エンタープライズグレードの AI 検索エンジンとして活用できます。

---

## アーキテクチャ（3段階パイプライン）

```
BigQuery テーブル
      ↓
[1] 取り込み & インデックス化
      チャンキング → ベクトル埋め込み変換
      ↓
[2] 検索 & 再ランキング
      クエリをベクトル化 → 意味的類似度マッチング
      ↓
[3] 合成 & グラウンディング
      Gemini に渡して根拠付き回答を生成（ハルシネーション抑制）
```

---

## 対応データストア種別

| 種別 | BigQuery 対応 | 備考 |
|---|---|---|
| 構造化データ | **対応（主要）** | テーブル行・JSON レコードを検索対象 |
| メディアデータ | **対応** | 動画・ニュース・ポッドキャスト等のメタデータ |
| 非構造化データ | **非対応** | Cloud Storage 経由のみ（PDF・HTML・DOCX等） |
| ウェブサイトデータ | 非対応 | クロール方式のみ |
| Healthcare FHIR | 非対応 | FHIR ストア経由のみ |

---

## 主なメリット

- **既存 BigQuery 資産を即活用** — 追加 ETL 不要
- **自然言語検索** — キーワードではなく意味ベースで検索
- **Gemini RAG との統合** — 根拠のある AI 回答を生成
- **定期自動同期** — 1日・3日・5日ごとに自動更新でデータ鮮度を維持
- **スキーマ自動検出** — BigQuery テーブル構造を自動推定
- **スケーラビリティ** — ペタバイト規模のデータに対応

---

## 取り込みモード

### モード比較

| モード | 方式 | 対象 | アクセス制御 |
|---|---|---|---|
| **ワンタイム（One-Time）** | 手動 | 単一テーブル | **対応** |
| **定期（Periodic）** | 自動（1・3・5日） | 複数テーブル | **非対応** |

### 調整（Reconciliation）モード

| モード | 動作 |
|---|---|
| **INCREMENTAL**（デフォルト） | 新規追加 + 同 ID の既存ドキュメントを更新（Upsert） |
| **FULL** | 完全再ベース。BigQuery に存在しないドキュメントは自動削除 |

---

## 主要機能

### スキーマ設定

- **自動検出（Auto-detect）**: インポート時にスキーマを自動推定
- **Dynamic モード**: 新フィールドが自動追加される
- **Static モード**: 事前定義フィールドのみ受け付ける
- **キープロパティのマッピング**: `title`・`description`・`uri`・`category` をマッピングすることで検索品質が向上

### フィールドレベルの設定

| プロパティ | 用途 |
|---|---|
| `retrievable` | 検索結果に表示（最大 50 フィールド） |
| `indexable` | フィルタ・ファセット・ブースト・ソートに使用 |
| `searchable` | テキスト検索対象 |
| `dynamicFacetable` | 動的ファセットとして利用可能 |
| `completable` | オートコンプリート候補の提供 |

### アクセス制御（ワンタイム取り込みのみ）

- ユーザー ID またはグループ ID によるドキュメントレベルのアクセス制御が可能
- ACL メタデータを BigQuery テーブルの `acl_info` 列として提供
- 対応 ID プロバイダー: Google Identity、Azure AD、Okta、Ping 等
- 1 ドキュメントあたり最大 **3,000 リーダー** まで設定可能

---

## セットアップ手順

### 前提条件

1. Google Cloud プロジェクトで **AI Applications API**（Discovery Engine API）を有効化
2. IAM 設定: Discovery Engine サービスアカウントに BigQuery 権限を付与
   - 同一プロジェクト: 自動付与
   - クロスプロジェクト: `BigQuery Job User` + `BigQuery Data Editor` ロールを手動付与
3. BigQuery テーブルのスキーマとデータを準備

### ワンタイム取り込み

1. Google Cloud コンソールで「AI Applications」→「データストア」→「データストアを作成」
2. データソースとして「**BigQuery**」を選択
3. データタイプ「**構造化データ**」を選択
4. 同期頻度で「**1 回**」を選択
5. BigQuery テーブルのパス（`project.dataset.table`）を指定
6. スキーマ設定: `title`・`description`・`uri` 等のキープロパティをマッピング
7. データストアのリージョン・表示名を設定し「作成」をクリック
8. ステータスが「インポート完了」になるまで待機

### 定期取り込み

1. 手順 1〜3 はワンタイムと同様
2. 同期頻度で「**定期的**」を選択し、1日・3日・5日のいずれかを指定
3. BigQuery データセットのパス（`project.dataset`）を指定
4. 同期対象のテーブルを選択（複数選択可）
5. コネクタ名・リージョンを設定し「作成」をクリック
6. 作成後 約 1 時間後に初回同期が自動実行され、以降は指定頻度で自動更新

### 検索アプリとの接続

1. 「アプリを作成」→「検索」を選択
2. 作成したデータストアをアプリに接続
3. コンソール・REST API・クライアントライブラリ（Python: `google-cloud-discoveryengine`）経由で検索クエリを実行

### Gemini グラウンディングとの統合（任意）

Vertex AI Studio または API から `VertexAiSearchTool` でデータストア ID を指定し、Gemini モデルの回答をデータストアで根拠付け（Grounding）する。

---

## ユースケース

| ユースケース | 内容 |
|---|---|
| 社内ナレッジベース検索 | 社内ドキュメント・製品情報・ポリシーを自然言語で検索 |
| カスタマーサポート | BigQuery の FAQ/サポート履歴を Gemini でグラウンディングしたチャットボット |
| EC・商品検索 | 商品カタログを自然言語で検索・パーソナライズレコメンド |
| 営業支援コパイロット | 契約データ・製品情報をリアルタイム検索して最適提案を生成 |
| メディア推薦 | 動画・ニュースのメタデータを元にコンテンツ推薦エンジンを構築 |

---

## 制限事項・考慮事項

| 制限 | 詳細 |
|---|---|
| 非構造化データ非対応 | PDF 等は BigQuery 経由不可、Cloud Storage 経由のみ |
| 権限の非継承 | BigQuery の権限はデータストアに引き継がれない |
| 定期取り込みのアクセス制御非対応 | Periodic Ingestion ではドキュメントレベルのアクセス制御が使用不可 |
| データストア数の上限 | 1アプリあたり最大 50 データストア |
| CMEK 制約 | 米国または EU のマルチリージョンのみ対応 |
| スキーマ後方互換性 | 非互換な変更はインポート失敗を招く |
| 処理時間 | データ規模によって数分〜数時間のインジェスト処理時間が必要 |
| エンティティデータストア制約 | Periodic Ingestion では各エンティティが単一 BigQuery テーブルにのみ対応 |

---

---

## Python を使ったセットアップ実装ガイド

> 以下は実際に `faqdata/data.jsonl`（22,794件のFAQ）を使って検証した手順と結果です。

### 検証環境

| 項目 | 値 |
|---|---|
| プロジェクト | `tuneyuki-gmail` |
| BigQuery データセット | `tuneyuki-gmail.vais_faq_demo` |
| BigQuery テーブル | `tuneyuki-gmail.vais_faq_demo.faq` |
| データストア | `bq-faq-demo-5ddfda06`（`global`リージョン）|
| ライブラリ | `google-cloud-bigquery==3.40.0` / `google-cloud-discoveryengine==0.13.12` |
| 認証 | Application Default Credentials（ADC）|

---

### 前提: IAM / API

#### 必要な API

```bash
# Discovery Engine API（AI Applications）
gcloud services enable discoveryengine.googleapis.com --project=PROJECT_ID

# BigQuery API（通常デフォルト有効）
gcloud services enable bigquery.googleapis.com --project=PROJECT_ID
```

#### 必要な権限

| 操作 | 必要なロール |
|---|---|
| BigQueryデータセット・テーブル作成 | `roles/bigquery.dataEditor` または `roles/bigquery.admin` |
| BigQueryデータ挿入 | `roles/bigquery.dataEditor` |
| VAIS データストア作成・インポート | `roles/discoveryengine.admin` |

> **注意**: サービスアカウントで操作する場合は上記ロールを付与すること。
> ADC（`gcloud auth application-default login`）を使うプロジェクトオーナー権限のユーザーアカウントでも動作確認済み。

---

### Step 1: BigQuery テーブルの準備

#### 1-1. スキーマ設計の注意点

`data_schema="custom"` を指定して BigQuery からインポートする場合、**ドキュメント ID として使う列の名前は `_id` でなければなりません**。`id` という列名では全件エラーになります。

```
エラー例: "Custom Document Id ('_id') was not found in document."
```

#### 1-2. テーブル作成コード

```python
from google.cloud import bigquery

PROJECT_ID = "your-project-id"
DATASET_ID = "vais_faq_demo"
TABLE_ID   = "faq"

client = bigquery.Client(project=PROJECT_ID)

# データセット作成
dataset = bigquery.Dataset(bigquery.DatasetReference(PROJECT_ID, DATASET_ID))
dataset.location = "US"
client.create_dataset(dataset, exists_ok=True)

# テーブル作成（_id が必須）
schema = [
    bigquery.SchemaField("_id",       "STRING", mode="REQUIRED"),  # ← _id が必須
    bigquery.SchemaField("question",  "STRING", mode="NULLABLE"),
    bigquery.SchemaField("answer",    "STRING", mode="NULLABLE"),
    bigquery.SchemaField("copyright", "STRING", mode="NULLABLE"),
    bigquery.SchemaField("url",       "STRING", mode="NULLABLE"),
]
table = bigquery.Table(
    bigquery.TableReference(
        bigquery.DatasetReference(PROJECT_ID, DATASET_ID), TABLE_ID
    ),
    schema=schema,
)
client.create_table(table, exists_ok=True)
```

> 既存テーブルの列名を変更する場合は ALTER TABLE で対応可能：
> ```sql
> ALTER TABLE `project.dataset.faq` RENAME COLUMN id TO _id;
> ```

#### 1-3. データの挿入

```python
import json, uuid

rows = []
with open("data.jsonl", encoding="utf-8") as f:
    for line in f:
        data = json.loads(line)
        rows.append({
            "_id":       uuid.uuid4().hex,
            "question":  data.get("Question", ""),
            "answer":    data.get("Answer", ""),
            "copyright": data.get("copyright", ""),
            "url":       data.get("url", ""),
        })

table_ref = f"{PROJECT_ID}.{DATASET_ID}.{TABLE_ID}"
# 1,000件ずつバッチ挿入（Streaming Insert の上限に配慮）
for i in range(0, len(rows), 1000):
    errors = client.insert_rows_json(table_ref, rows[i:i+1000])
    if errors:
        print(f"Errors: {errors[:3]}")
```

**実測値**: 22,794件を1,000件バッチで挿入 → 全23バッチ、エラーなし

---

### Step 2: データストア＆エンジンの作成

```python
import uuid
from google.cloud import discoveryengine

PROJECT_ID = "your-project-id"
LOCATION   = "global"

def create_data_store(display_name: str) -> str:
    client = discoveryengine.DataStoreServiceClient()
    parent = client.collection_path(PROJECT_ID, LOCATION, "default_collection")

    data_store_id = f"my-store-{uuid.uuid4().hex[:8]}"

    data_store = discoveryengine.DataStore(
        display_name=display_name,
        industry_vertical=discoveryengine.IndustryVertical.GENERIC,
        content_config=discoveryengine.DataStore.ContentConfig.NO_CONTENT,
    )

    operation = client.create_data_store(
        request=discoveryengine.CreateDataStoreRequest(
            parent=parent,
            data_store=data_store,
            data_store_id=data_store_id,
        )
    )
    response = operation.result()   # LRO 完了まで同期待機
    return data_store_id


def create_engine(data_store_id: str, display_name: str) -> str:
    client = discoveryengine.EngineServiceClient()
    parent = client.collection_path(PROJECT_ID, LOCATION, "default_collection")

    engine_id = f"my-engine-{uuid.uuid4().hex[:8]}"

    engine = discoveryengine.Engine(
        display_name=display_name,
        solution_type=discoveryengine.SolutionType.SOLUTION_TYPE_SEARCH,
        search_engine_config=discoveryengine.Engine.SearchEngineConfig(
            search_tier="SEARCH_TIER_ENTERPRISE",
            search_add_ons=["SEARCH_ADD_ON_LLM"],
        ),
        data_store_ids=[data_store_id],
    )

    operation = client.create_engine(
        request=discoveryengine.CreateEngineRequest(
            parent=parent,
            engine=engine,
            engine_id=engine_id,
        )
    )
    response = operation.result()
    return engine_id
```

---

### Step 3: BigQuery → VAIS ワンタイム（One-Time）取り込み

```python
import time
from google.cloud import discoveryengine

def import_from_bigquery(
    data_store_id: str,
    bq_project: str,
    bq_dataset: str,
    bq_table: str,
    reconciliation_mode: str = "FULL",
) -> dict:
    client = discoveryengine.DocumentServiceClient()
    parent = client.branch_path(PROJECT_ID, LOCATION, data_store_id, "default_branch")

    mode = (
        discoveryengine.ImportDocumentsRequest.ReconciliationMode.FULL
        if reconciliation_mode == "FULL"
        else discoveryengine.ImportDocumentsRequest.ReconciliationMode.INCREMENTAL
    )

    bq_source = discoveryengine.BigQuerySource(
        project_id=bq_project,
        dataset_id=bq_dataset,
        table_id=bq_table,
        data_schema="custom",   # GENERIC データストアでのみ使用可能
    )

    start = time.time()
    operation = client.import_documents(
        request=discoveryengine.ImportDocumentsRequest(
            parent=parent,
            bigquery_source=bq_source,
            reconciliation_mode=mode,
        )
    )

    print(f"Operation: {operation.operation.name}")
    response = operation.result()   # 完了まで同期待機
    elapsed = time.time() - start

    return {
        "elapsed_seconds": round(elapsed, 1),
        "error_samples": [str(e) for e in response.error_samples],
    }
```

#### 調整モード（Reconciliation Mode）の使い分け

| モード | 動作 | 用途 |
|---|---|---|
| `FULL` | BQ テーブルの内容で完全置換（BQにないドキュメントを削除） | 初回取り込み・全件更新 |
| `INCREMENTAL` | 新規追加・同`_id`の更新のみ（既存を削除しない） | 差分更新 |

---

### 実行ログ（実測）

#### データストア・エンジン作成

```
Creating Data Store: bq-faq-demo-5ddfda06
Waiting for data store creation...
Data Store created: projects/382817498526/.../dataStores/bq-faq-demo-5ddfda06

Creating Engine: bq-faq-engine-9ca2c13a
Waiting for engine creation...
Engine created: projects/382817498526/.../engines/bq-faq-engine-9ca2c13a
```

#### ワンタイム取り込み（22,794件 / FULL モード）

```
Starting import: tuneyuki-gmail.vais_faq_demo.faq -> bq-faq-demo-5ddfda06
Operation started: .../operations/import-documents-1897088553642479759
Waiting for import to complete...

Import completed in 413.0s   # 約7分
No errors reported.
```

#### ドキュメント数確認

```
Total documents: 22794   # BQ テーブルの行数と一致
struct_data keys: ['answer', 'copyright', 'url', 'question']
```

---

### よくあるエラーと対処

#### `Custom Document Id ('_id') was not found in document.`

- **原因**: `data_schema="custom"` の場合、BigQuery のドキュメント ID 列名は必ず `_id` でなければならない。`id` という名前では全件失敗する。
- **対処**: `ALTER TABLE ... RENAME COLUMN id TO _id` で修正するか、最初から `_id` という列名でテーブルを作成する。

#### `Permission 'discoveryengine.dataStores.create' denied`

- **原因**: サービスアカウントに `discoveryengine.admin`（または同等）のロールが付与されていない。
- **対処**: `roles/discoveryengine.admin` を付与する。
  ```bash
  gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:SA_EMAIL" \
    --role="roles/discoveryengine.admin"
  ```

#### `Access Denied: ... bigquery.datasets.create permission`

- **原因**: サービスアカウントに BigQuery の書き込み権限がない。
- **対処**: `roles/bigquery.dataEditor` または `roles/bigquery.admin` を付与する。

---

## グラウンディングメタデータ検証結果（BQ 取り込み vs オリジナル）

> `faqdata/data.jsonl`（22,794件）を3種類のデータストアで ADK `VertexAiSearchTool` グラウンディングを実行して比較した結果。

### 検証ストア構成

| ストア | データソース | BQカラム名 | VAIS スキーマ自動検出結果 |
|---|---|---|---|
| **bq_original** | BQ テーブル直接 | `_id`, `question`, `answer`, `url`, `copyright` | `question: keyPropertyMapping=question` / `url: searchable+indexable` |
| **bq_view** | BQ ビュー（リネーム済み） | `_id`, `title`, `answer`, `uri`, `copyright` | `title: keyPropertyMapping=title` / `uri: keyPropertyMapping=uri` |
| **original** | GCS 経由 JSONL | トップレベル `_id`, `uri`, `title`, `structData` | `uri` / `title` がドキュメントレベルに格納 |

### 比較結果（3クエリ × 3ストア）

| ストア | chunks数 | `uri` あり | `title` あり | URL 一致 |
|---|---|---|---|---|
| bq_original (question/url) | 30〜63 | ✅ | ❌ | ✅ |
| bq_view (title/uri)        | 32〜47 | ✅ | ✅ | ✅ |
| original (JSONL形式)        | 26〜45 | ✅ | ✅ | ✅ |

### 詳細考察

#### bq_original（`question`/`url` カラム名）

```
retrieved_context.uri   = "https://www.jbaudit.go.jp/..."   ← 正しく取得
retrieved_context.title = None                               ← 空
retrieved_context.text  = 'answer: "公共工事は..."'          ← struct_data の生テキスト
```

- `url` という名前は VAIS が URI として自動認識し `retrieved_context.uri` に反映される
- `question` は `keyPropertyMapping=question`（カスタム名）として struct_data に残り、セマンティック検索のコンテンツとして使われる → **検索は正常動作**
- ただし `keyPropertyMapping` が `"title"` でないため `retrieved_context.title` は空

#### bq_view（`title`/`uri` カラム名）

```
retrieved_context.uri   = "https://www.jbaudit.go.jp/..."   ← 正しく取得
retrieved_context.title = "公共工事についての質問..."         ← 正しく取得
retrieved_context.text  = 'answer: "..."'                   ← struct_data の生テキスト
```

- `title` カラム → `keyPropertyMapping=title` → `retrieved_context.title` に反映される ✅
- `uri` カラム → `keyPropertyMapping=uri` → `retrieved_context.uri` に反映される ✅
- BQ VIEW を使っても、カラム名を `title`/`uri` にすれば両方のメタデータを取得できる
- **注意**: データストアだけでなく**エンジンも必ず作成**すること（エンジン未作成だと検索結果がゼロになる）

#### original（GCS 経由 JSONL、標準フォーマット）

```
retrieved_context.uri   = "https://..."   ✅
retrieved_context.title = "公共工事につい..."  ✅
retrieved_context.text  = 'structData:\n  Answer: "..."'
```

- JSONL のトップレベルに `uri`/`title` を置く標準フォーマットでも同様に両方取得できる

### 結論：BQ 取り込み時のカラム命名とグラウンディングメタデータ

| カラム名 | `keyPropertyMapping` | `retrieved_context.uri` | `retrieved_context.title` |
|---|---|---|---|
| `url` | 自動: `searchable+indexable`（URI 認識なし） | ✅（自動認識） | ❌ |
| `question` | `keyPropertyMapping=question`（カスタム） | ❌ | ❌ |
| `uri` | `keyPropertyMapping=uri` | ✅ | ❌ |
| `title` | `keyPropertyMapping=title` | ❌ | ✅ |

**BQ 取り込みで `uri` と `title` を両方 `retrieved_context` に反映させる方法**:

BQ テーブル（またはビュー）のカラム名を `_id` / `title` / `uri` にして取り込む。
BQ VIEW を使えば元テーブルを変えずに列名を変えられる：

```sql
CREATE OR REPLACE VIEW `project.dataset.faq_for_vais` AS
SELECT
  id      AS _id,
  question AS title,   -- → retrieved_context.title
  url      AS uri,     -- → retrieved_context.uri
  answer,
  copyright
FROM `project.dataset.faq`;
```

**重要: `keyPropertyMapping` は設定後に変更不可**（API エラー: `"Schema update cannot alter the key property mapping"`）。
一度インポートしたデータストアのマッピングを変えたい場合は、新しいデータストアを作り直す必要がある。

---

### スクリプト参照

実際に動かした完全なスクリプトは `docs/scripts/` に格納：

| ファイル | 内容 |
|---|---|
| [`docs/scripts/bigquery_setup.py`](scripts/bigquery_setup.py) | BQ データセット・テーブル作成、JSONL データロード |
| [`docs/scripts/bq_to_vais.py`](scripts/bq_to_vais.py) | データストア・エンジン作成、BQ からのワンタイム取り込み |
| `test_grounding_bq.py` | BQ 取り込みストアのグラウンディング検証（`--compare` でオリジナルと比較） |
| `test_3stores.py` | 3ストア比較グラウンディングテスト |

#### 実行例

```bash
# 1. BQ テーブル作成 & データロード
python docs/scripts/bigquery_setup.py create
python docs/scripts/bigquery_setup.py load faqdata/data.jsonl
python docs/scripts/bigquery_setup.py verify

# 2. VAIS データストア・エンジン作成 & ワンタイム取り込み（ワンコマンド）
python docs/scripts/bq_to_vais.py full_setup
# → bq_vais_config.json に data_store_id / engine_id を保存

# 3. 再インポート（既存データストアへ）
python docs/scripts/bq_to_vais.py import <data_store_id> FULL

# 4. インポート結果確認
python docs/scripts/bq_to_vais.py verify <data_store_id>
```

---

## データストアとエンジンの関係

### エンジンとは何か

Vertex AI Search では **データストア（Data Store）** と **エンジン（Engine）** は別リソースとして分離されている。

```
[BigQuery / JSONL / Cloud Storage]
         ↓ インポート
┌────────────────────────┐
│      Data Store        │  ドキュメントを保存・インデックス化する場所
│  (bq-faq-demo-xxxx)   │  例: 図書館の書棚
└────────────────────────┘
         ↓ アタッチ
┌────────────────────────┐
│        Engine          │  検索・推論をサーブするエンドポイント
│  (bq-faq-engine-xxxx) │  例: 図書館の司書カウンター
└────────────────────────┘
         ↓ クエリ
   ADK VertexAiSearchTool / Gemini グラウンディング
```

| リソース | 役割 | API パス |
|---|---|---|
| **Data Store** | ドキュメントの保存・インデックス管理 | `collections/default_collection/dataStores/{id}` |
| **Engine** | 検索サービスのエンドポイント（Serving Config を内包） | `collections/default_collection/engines/{id}` |

### データストアだけでは検索できない

**エンジンがない状態での検索結果は 0 件（エラーなし）**。

ADK の `VertexAiSearchTool` は `data_store_id` を渡すが、内部的には Engine の `servingConfigs/default_config` エンドポイントへ検索リクエストを投げる。Engine がない場合、エンドポイントが存在しないため検索結果が静かに 0 件になる。

### 実際に踏んだ問題

`bq-title-view-b894700a`（BQ VIEW から作成したデータストア）でエンジン作成を忘れた結果、グラウンディングが 0 件になった。

```
bq_view → chunks=0, uri=False, title=False  ← エンジン未作成のため
```

エンジン作成後 60 秒ほどで検索が有効になり、正常に grounding metadata が取得できた。

```python
# エンジンの作成（bq_to_vais.py より）
def create_engine(data_store_id: str, display_name: str = "BQ FAQ Engine") -> str:
    client = discoveryengine.EngineServiceClient()
    parent = client.collection_path(PROJECT_ID, LOCATION, "default_collection")
    engine_id = f"bq-faq-engine-{uuid.uuid4().hex[:8]}"

    engine = discoveryengine.Engine(
        display_name=display_name,
        solution_type=discoveryengine.SolutionType.SOLUTION_TYPE_SEARCH,
        data_store_ids=[data_store_id],
        search_engine_config=discoveryengine.Engine.SearchEngineConfig(
            search_tier=discoveryengine.SearchTier.SEARCH_TIER_ENTERPRISE,
        ),
    )
    operation = client.create_engine(parent=parent, engine=engine, engine_id=engine_id)
    result = operation.result()
    return result.name.split("/")[-1]
```

> **チェックポイント**: データストア作成後、必ずエンジンを作成すること。コンソール UI では「アプリ作成」がエンジン作成に相当する。

---

## copyright フィルタ検証

### 検証内容

3ストアで `copyright` フィールドを使った検索フィルタが動作するか確認した（2026-02-22 実施）。

### スキーマ確認

全ストアの `copyright` フィールドは以下のアノテーションを持つ（`filterable` は未設定）：

```json
{
  "type": "string",
  "indexable": true,
  "searchable": true,
  "retrievable": true,
  "dynamicFacetable": true
}
```

### フィルタ構文の違い

| 構文 | 例 | 結果 |
|---|---|---|
| `:` 演算子 | `copyright: "厚生労働省"` | ❌ 全ストアでエラー（`Unsupported field on ":" operator`） |
| `ANY()` | `copyright: ANY("厚生労働省")` | BQ ストアは ✅、original は ⚠️（後述） |

**正しい構文は `ANY()` 形式**。`indexable: true` があれば `filterable: true` なしでも `ANY()` が使える。

### フィールドパスの違い（重要）

インポート方式によってフィールドのパス構造が異なる：

| インポート方式 | フィールド格納場所 | フィルタパス |
|---|---|---|
| BigQuery | トップレベル | `copyright: ANY("値")` |
| JSONL（Cloud Storage 等） | `structData` 配下 | `structData.copyright: ANY("値")` |

### 検証結果

| ストア | フィルタ条件 | 結果 |
|---|---|---|
| `bq_original` | `copyright: ANY("厚生労働省")` | ✅ 3件 |
| `bq_view` | `copyright: ANY("厚生労働省")` | ✅ 3件 |
| `original`（JSONL） | `copyright: ANY("厚生労働省")` | ❌ 0件（パスが違う） |
| `original`（JSONL） | `structData.copyright: ANY("厚生労働省")` | ✅ 3件 |

### 検証コード

```python
from google.cloud import discoveryengine

client = discoveryengine.SearchServiceClient()
serving_config = (
    f"projects/{PROJECT_ID}/locations/global/collections/default_collection"
    f"/dataStores/{DS_ID}/servingConfigs/default_config"
)

# BQ インポートのストア → トップレベルパス
response = client.search(discoveryengine.SearchRequest(
    serving_config=serving_config,
    query="農業",
    page_size=5,
    filter='copyright: ANY("厚生労働省")',
))

# JSONL インポートのストア → structData. プレフィックスが必要
response = client.search(discoveryengine.SearchRequest(
    serving_config=serving_config,
    query="農業",
    page_size=5,
    filter='structData.copyright: ANY("厚生労働省")',
))
```

> **注意**: `struct_data.copyright`（スネークケース）はエラーになる。`structData.copyright`（キャメルケース）が正しい。

---

## 参考資料

- [Create a search data store | Vertex AI Search](https://cloud.google.com/generative-ai-app-builder/docs/create-data-store-es)
- [About apps and data stores | Vertex AI Search](https://cloud.google.com/generative-ai-app-builder/docs/create-datastore-ingest)
- [Provide or auto-detect a schema | Vertex AI Search](https://cloud.google.com/generative-ai-app-builder/docs/provide-schema)
- [Refresh structured and unstructured data | Vertex AI Search](https://cloud.google.com/generative-ai-app-builder/docs/refresh-data)
- [Use data source access control | Vertex AI Search](https://cloud.google.com/generative-ai-app-builder/docs/data-source-access-control)
- [Grounding with Vertex AI Search | Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs/grounding/grounding-with-vertex-ai-search)
