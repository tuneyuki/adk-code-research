# Artifact (アーティファクト) について

ADK における **Artifact** とは、Agent や Tool が生成・利用する「テキスト以外のファイルやデータ」を指します。
ユーザーがアップロードしたファイルや、Agent がコード実行によって生成した画像（グラフなど）がこれに該当します。

## 1. 役割と保管場所

Artifact は `ArtifactService` によって一元管理されます。
保存先は設定により切り替え可能です：
*   **FileArtifactService**: ローカルファイルシステムに保存。
    *   パス構成: `root/users/{user_id}/sessions/{session_id}/artifacts/{filename}`
*   **GcsArtifactService**: Google Cloud Storage に保存。
*   **InMemoryArtifactService**: メモリ上に一時保存。

## 2. 生成からレスポンスまでの流れ (画像生成の例)

「Agent が Python コードを実行して画像を生成した」場合の流れは以下の通りです。

1.  **生成**:
    *   `CodeExecutor` がコードを実行し、出力ファイル（例: `plot.png`）や、インライン画像データを検出します。
2.  **保存**:
    *   `InvocationContext.artifact_service.save_artifact()` が呼ばれます。
    *   ファイルの実体が Storage (Disk/GCS) に書き込まれます。
    *   「バージョン番号 (version)」が発行されます。
3.  **レスポンスへの埋め込み**:
    *   Agent が返す `Event` オブジェクト内の `actions.artifact_delta` に、「ファイル名」と「バージョン」のマッピングが記録されます。
    *   **重要**: LLMのテキストレスポンス自体には、ファイルのバイナリデータは含まれません。代わりに `Saved as artifact: plot.png` のようなプレースホルダーテキストに置き換わることがあります。
4.  **クライアント側の処理**:
    *   API の呼び出し元（Web UI や CLI）は、受け取った `Event` の `artifact_delta` を見て、「新しいファイルが生成された」ことを知ります。
    *   利用者は、別途 API (例: `/artifacts/{filename}`) を通じてファイルの実体をダウンロード・表示します。

## 3. リンクやパスは返ってくるのか？

**直接的な URL (http://...) が LLM のテキスト回答として返ってくるわけではありません。**

*   **構造化データとして返却**: レスポンス（`Event`）のメタデータ部分 (`artifact_delta`) に、「ファイル名」と「バージョン情報」が含まれて返ってきます。
*   **パスの解決**: 実際にアクセスするためのパス（URI）は、`ArtifactService` の内部で管理されています（`canonical_uri`）。
    *   ローカルの場合: `file:///path/to/...`
    *   GCSの場合: `gs://bucket/...`
*   アプリケーション（Web UI等）は、このメタデータを使って適切なリンクを生成し、ユーザーに表示する仕組みになっています。

## まとめ

*   **Artifact** = 生成されたファイル。
*   **保存** = `ArtifactService` が裏で勝手にやってくれる。
*   **リターン** = ファイルそのものではなく、「ファイル名とバージョン情報」がメタデータとして返ってくる。
