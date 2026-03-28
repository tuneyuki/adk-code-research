# ADK Skills アーキテクチャ詳細解説

> **対象バージョン**: google-adk 1.27.2
> **ステータス**: experimental（`FeatureName.SKILL_TOOLSET`）

## 概要

ADK Skills は、エージェントの能力を**動的に拡張**するための仕組みです。事前に定義された「スキルフォルダ」をエージェントに登録しておくと、エージェントがユーザーのクエリに応じて**必要なスキルを自分で選択・読み込み・実行**します。

コアとなるアイデアは「**Lazy Loading**」です。全スキルの指示を最初からプロンプトに含めるのではなく、スキルのメタデータ（名前・説明）のみをシステムプロンプトに含め、エージェントが必要と判断したスキルだけを `load_skill` ツールで読み込みます。

---

## 1. スキルのディレクトリ構造

```
skills/
  my-skill/                     # ディレクトリ名 = スキル名（kebab-case）
    SKILL.md                    # 必須: フロントマター + 指示書
    references/                 # 任意: 追加ドキュメント
      api-guide.md
    assets/                     # 任意: テンプレート、スキーマ等
      template.json
      image.png                 # バイナリファイルも可
    scripts/                    # 任意: 実行可能スクリプト
      setup.py
      deploy.sh
```

### SKILL.md のフォーマット

```markdown
---
name: my-skill
description: このスキルが何をするか、いつ使うべきかの説明
license: MIT
compatibility: google-adk>=1.27.0
allowed-tools: tool_a tool_b
metadata:
  adk_additional_tools:
    - my_custom_tool
    - another_tool
---

# ここからが L2 Instructions（スキルの詳細な指示書）

エージェントがこのスキルを load_skill で読み込んだときに返される内容。
ステップバイステップの手順や、判断基準を記述する。
```

---

## 2. データモデル（3層構造: L1 / L2 / L3）

Skills は意図的に3層に分けてロードされます。

| レイヤー | モデル | 内容 | ロードタイミング |
|---|---|---|---|
| **L1** | `Frontmatter` | `name`, `description`, `license`, `compatibility`, `allowed_tools`, `metadata` | SkillToolset 初期化時 |
| **L2** | `instructions` (str) | SKILL.md の本文（フロントマター以下の Markdown） | `load_skill` ツール呼び出し時 |
| **L3** | `Resources` | `references/`, `assets/`, `scripts/` ディレクトリの内容 | `load_skill_resource` / `run_skill_script` ツール呼び出し時 |

### ソースコード上の定義 (`skills/models.py`)

```python
class Frontmatter(BaseModel):     # L1: メタデータ
    name: str                     # kebab-case, 最大64文字
    description: str              # 最大1024文字
    license: Optional[str]
    compatibility: Optional[str]  # 最大500文字
    allowed_tools: Optional[str]  # スペース区切りのツール名
    metadata: dict[str, Any]      # adk_additional_tools 等

class Script(BaseModel):          # スクリプトラッパー
    src: str

class Resources(BaseModel):      # L3: リソース
    references: dict[str, str | bytes]
    assets: dict[str, str | bytes]
    scripts: dict[str, Script]

class Skill(BaseModel):           # L1 + L2 + L3 の統合モデル
    frontmatter: Frontmatter
    instructions: str             # L2
    resources: Resources          # L3
```

**バリデーション**:
* `name`: kebab-case 正規表現 `^[a-z0-9]+(-[a-z0-9]+)*$`、64文字以下
* `name` はディレクトリ名と一致しなければならない
* `metadata.adk_additional_tools` は `list` 型である必要がある

---

## 3. スキルの読み込み方法

### 3.1 ローカルファイルシステムから

```python
from google.adk.skills import load_skill_from_dir, list_skills_in_dir

# 単一スキルの読み込み
skill = load_skill_from_dir("./skills/my-skill")

# ディレクトリ内の全スキルを一覧取得（Frontmatter のみ）
skills_map = list_skills_in_dir("./skills")
# => {"my-skill": Frontmatter(...), "other-skill": Frontmatter(...)}
```

### 3.2 GCS (Google Cloud Storage) から

```python
from google.adk.skills import load_skill_from_gcs_dir, list_skills_in_gcs_dir

# GCS からスキル一覧を取得
skills_map = list_skills_in_gcs_dir(
    bucket_name="my-bucket",
    skills_base_path="path/to/skills"
)

# GCS から単一スキルを読み込み
skill = load_skill_from_gcs_dir(
    bucket_name="my-bucket",
    skill_id="my-skill",
    skills_base_path="path/to/skills"
)
```

### 3.3 内部の SKILL.md パース処理 (`skills/_utils.py`)

```
SKILL.md の内容:
  "---"  →  フロントマター開始
  YAML   →  yaml.safe_load() → dict
  "---"  →  フロントマター終了
  本文   →  instructions (str)
```

1. `_parse_skill_md_content()`: 生テキストから `(dict, body)` を分離
2. `Frontmatter.model_validate(parsed)`: Pydantic バリデーション
3. `_load_dir()`: `references/`, `assets/`, `scripts/` を再帰的に読み込み
4. ディレクトリ名とフロントマターの `name` の一致を検証

---

## 4. SkillToolset とツール群

`SkillToolset` は `BaseToolset` を継承し、4つのツールを LLM に提供します。

### 4.1 ツール一覧

| ツール名 | クラス | 役割 |
|---|---|---|
| `list_skills` | `ListSkillsTool` | 登録済みスキルの一覧を XML 形式で返す |
| `load_skill` | `LoadSkillTool` | 指定スキルの L2 instructions を読み込む |
| `load_skill_resource` | `LoadSkillResourceTool` | スキル内の L3 リソースファイルを読み込む |
| `run_skill_script` | `RunSkillScriptTool` | スキル内のスクリプトを実行する |

### 4.2 SkillToolset の初期化

```python
from google.adk.tools.skill_toolset import SkillToolset
from google.adk.skills import load_skill_from_dir

skills = [
    load_skill_from_dir("./skills/my-skill"),
    load_skill_from_dir("./skills/other-skill"),
]

toolset = SkillToolset(
    skills=skills,
    code_executor=my_code_executor,       # スクリプト実行に必要
    script_timeout=300,                    # シェルスクリプトのタイムアウト（秒）
    additional_tools=[my_custom_tool],     # スキルから動的に有効化されるツール候補
)
```

### 4.3 エージェントへの組み込み

```python
agent = LlmAgent(
    name="my-agent",
    model="gemini-2.0-flash",
    tools=[toolset],    # SkillToolset を tools に渡す
)
```

---

## 5. 実行時フロー（シーケンス）

```
ユーザー: "レポートを作って"
    │
    ▼
┌─────────────────────────────────────────┐
│ SkillToolset.process_llm_request()      │
│ → システムプロンプトに以下を注入:        │
│   1. _DEFAULT_SKILL_SYSTEM_INSTRUCTION   │
│   2. <available_skills> XML              │
│      (全スキルの name + description)     │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ LLM が判断:                              │
│ "report-generator スキルが関連しそう"    │
│ → list_skills() で一覧確認（任意）       │
│ → load_skill(name="report-generator")   │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ LoadSkillTool.run_async()               │
│ → L2 instructions を返す                │
│ → ステートに記録:                        │
│   state["_adk_activated_skill_{agent}"] │
│   = ["report-generator"]                │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ LLM が instructions に従って行動:        │
│ → load_skill_resource() でテンプレ取得  │
│ → run_skill_script() でスクリプト実行   │
│ → adk_additional_tools で追加ツール使用 │
└─────────────────────────────────────────┘
```

### 5.1 システムプロンプト注入の詳細

`SkillToolset.process_llm_request()` は **毎回の LLM リクエスト前**に呼ばれ、以下をシステムプロンプトに追加します:

1. **`_DEFAULT_SKILL_SYSTEM_INSTRUCTION`**: スキルの使い方の一般指示
2. **`<available_skills>` XML**: 全スキルの名前と説明をXMLで列挙

```xml
<available_skills>
<skill>
<name>report-generator</name>
<description>Generates formatted reports from data</description>
</skill>
<skill>
<name>data-analyzer</name>
<description>Analyzes datasets and provides insights</description>
</skill>
</available_skills>
```

### 5.2 スキルのアクティベーションと追加ツールの動的解決

`load_skill` が呼ばれると、ステートに `_adk_activated_skill_{agent_name}` キーでスキル名が記録されます。

次の LLM リクエスト時、`SkillToolset.get_tools()` → `_resolve_additional_tools_from_state()` が実行され:

1. アクティベートされたスキルの `metadata.adk_additional_tools` を収集
2. `SkillToolset` の `additional_tools` から該当するツールを解決
3. 解決されたツールをツール一覧に追加

```python
# SKILL.md のフロントマター
metadata:
  adk_additional_tools:
    - web_search        # このスキルがロードされると web_search ツールが使えるようになる
    - file_writer

# SkillToolset 初期化時に候補を渡しておく
SkillToolset(
    skills=[...],
    additional_tools=[web_search_tool, file_writer_tool, unused_tool],
    # → web_search と file_writer はスキルがアクティベートされたときのみ LLM に公開
)
```

---

## 6. スクリプト実行の仕組み

### 6.1 `RunSkillScriptTool` → `_SkillScriptCodeExecutor`

スクリプト実行は `_SkillScriptCodeExecutor` に委譲されます。

**処理フロー**:
1. スキルの全リソース（references, assets, scripts）を `_build_wrapper_code()` で Python コードに埋め込む
2. `tempfile.TemporaryDirectory()` に全ファイルを展開
3. スクリプト種別に応じて実行:
   * **`.py`**: `runpy.run_path()` で実行（`sys.argv` に引数をセット）
   * **`.sh` / `.bash`**: `subprocess.run()` で bash 実行（JSON エンベロープで stdout/stderr を返す）
4. 結果を `{"stdout", "stderr", "status"}` で返す

### 6.2 CodeExecutor の解決順序

1. `SkillToolset` に渡された `code_executor`
2. エージェントの `agent.code_executor`（フォールバック）
3. どちらもない場合はエラー `NO_CODE_EXECUTOR`

### 6.3 ペイロードサイズ制限

* `_MAX_SKILL_PAYLOAD_BYTES = 16 MB`: スキル全リソースの合計がこれを超えると警告

---

## 7. BashTool（`execute_bash`）

`ExecuteBashTool` は Skills と連携して使われることを想定した、独立した bash コマンド実行ツールです。

```python
from google.adk.tools.bash_tool import ExecuteBashTool, BashToolPolicy

bash_tool = ExecuteBashTool(
    workspace=pathlib.Path("./workspace"),
    policy=BashToolPolicy(
        allowed_command_prefixes=("ls", "cat", "grep"),  # 許可するコマンド
    ),
)
```

**特徴**:
* **常にユーザー確認を要求**: `tool_context.request_confirmation()` が呼ばれ、ユーザーが承認しない限り実行されない
* **コマンドプレフィックスによるフィルタリング**: `BashToolPolicy.allowed_command_prefixes` で許可するコマンドを制限（デフォルトは `("*",)` で全許可）
* **タイムアウト**: 30秒固定
* **shell=False**: `shlex.split()` でコマンドを分割し、シェルインジェクションを防止

---

## 8. バイナリファイルの「コンテンツインジェクション」パターン

`LoadSkillResourceTool` がバイナリファイル（画像、PDF等）を検出した場合:

1. ツールレスポンスとして `"Binary file detected. The content has been injected into the conversation history for you to analyze."` を返す
2. 次の LLM リクエスト時、`process_llm_request()` のオーバーライドが:
   * レスポンスに含まれる `_BINARY_FILE_DETECTED_MSG` を検出
   * 元のバイナリコンテンツを取得
   * `types.Part(inline_data=types.Blob(...))` として `llm_request.contents` に追加

これにより、LLM はバイナリコンテンツ（画像解析等）を直接参照できます。

---

## 9. 設計上の重要なポイント

### 9.1 なぜ Lazy Loading なのか

* **トークン効率**: 全スキルの instructions をプロンプトに含めるとトークンが膨大になる。L1 の name/description だけなら最小限。
* **スケーラビリティ**: スキル数が増えても、LLM が見る情報量は制御可能。
* **コンテキスト汚染の防止**: 不要なスキルの指示が LLM の判断に悪影響を与えない。

### 9.2 ステートによるアクティベーション管理

* `_adk_activated_skill_{agent_name}` キーでどのスキルがアクティブかを管理
* これにより `adk_additional_tools` の動的解決がステートフルに行われる
* エージェント名ごとに管理されるため、マルチエージェント構成でも衝突しない

### 9.3 ToolUnion としての柔軟性

`SkillToolset` の `additional_tools` は `list[ToolUnion]` を受け取り、以下の全てに対応:
* `BaseTool` インスタンス
* `BaseToolset` インスタンス（内部のツールが展開される）
* `callable`（`FunctionTool` にラップされる）

---

## 10. 仕様準拠

フロントマターの `allowed_tools` フィールドは [Agent Skills 仕様](https://agentskills.io/specification#allowed-tools-field) に準拠しています。ADK 固有の拡張として `metadata.adk_additional_tools` が使われます。
