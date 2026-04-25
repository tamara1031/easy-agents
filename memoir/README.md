# memoir — ChromaDB ベースのベクター長期記憶パッケージ

ChromaDB (Docker) を使ったベクトル検索型の長期記憶管理パッケージです。ナレッジの保存・検索・更新・削除を自然言語クエリで行え、Claude Code スキルとして統合されています。

---

## 前提条件

| 要件 | 詳細 |
| :--- | :--- |
| Docker | Docker Desktop / Docker Engine / Colima いずれか起動済みであること |
| Python | 3.10 以上 |

> `chromadb` Python パッケージおよびエンベディングモデルは初回実行時に自動インストール・ダウンロードされます。

---

## インフラ仕様

| 項目 | 値 |
| :--- | :--- |
| Vector DB | ChromaDB 0.6.3 (Docker) |
| コンテナ名 | `copilot-memory-chromadb` |
| ポート | `18000` (ホスト) → `8000` (コンテナ内) |
| 永続ボリューム | `~/.local/share/copilot-memory/chroma-data` |
| 埋め込みモデル | `all-MiniLM-L6-v2` (クライアントサイド ONNX) |
| 距離関数 | cosine |
| コレクション名 | `long_term_memory` |
| ヘルスチェック | `/api/v2/heartbeat` (fallback: `/api/v1/heartbeat`) |
| 再起動ポリシー | `unless-stopped` |

ボリュームはシステム共通パス (`~/.local/share/copilot-memory/`) に永続化されるため、ワークスペースが変わっても同一の DB を参照し続けます。

---

## クイックスタート

> **作業ディレクトリ**: 以下のコマンドはすべて `memoir/` ディレクトリで実行してください。スクリプトパスが相対指定です。
> ```bash
> cd /path/to/easy-agents-hub/memoir
> ```

### 1. ChromaDB コンテナを起動

```bash
docker compose -f skills/long-term-memory/docker/docker-compose.yml up -d
```

### 2. 起動確認

```bash
# HTTP でヘルスチェック (ChromaDB 0.6.3 以降は v2 を使用。失敗時は v1 に自動フォールバック)
curl http://localhost:18000/api/v2/heartbeat
# 成功時のレスポンス例: {"nanosecond heartbeat": 1745123456789000000}
# 失敗時 (コンテナ未起動): curl: (7) Failed to connect to localhost port 18000

# v2 が使えない旧バージョンの場合:
# curl http://localhost:18000/api/v1/heartbeat
```

`chromadb` Python パッケージは初回スクリプト実行時に自動インストールされます（`pip install` 不要）。

> **自動起動について**: スクリプトは `_common.py` を通じて Docker が未起動の場合に `docker compose up -d` を自動実行しますが、初回セットアップは手動起動を推奨します（Docker デーモン自体が未起動の場合は自動起動も失敗するため）。

---

## Claude Code からの利用

`long-term-memory` スキルは APM パッケージとして提供されており、Claude Code エージェントが自動的に判断して呼び出します。

**明示的に呼び出す場合**: 会話の中で以下のような自然言語フレーズを使うと Claude Code がスキルを起動します。

| 操作 | 例 |
| :--- | :--- |
| 保存 | 「この設定を覚えておいて」「後で参照できるように保存して」 |
| 検索 | 「前に話したデプロイ手順を思い出して」「○○について記憶を検索して」 |
| 削除 | 「その情報は古いので忘れて」 |

**Claude Code の動作フロー**:
```
ユーザー: "この設定を覚えておいて"
→ Claude Code が long-term-memory スキルを検出
→ memory_save.py を実行して ChromaDB に保存
→ "保存しました (ID: xxxx)" と返答
```

`skills/long-term-memory/SKILL.md` は Claude Code がスキルを認識するための機械可読な定義ファイルです。いつ・どのスクリプトを呼ぶかの判断基準が記述されています。

---

## 操作一覧

### save — ナレッジの保存

ユーザーの入力を独立したナレッジ単位に分解してから保存することを推奨します。

| 引数 | 説明 |
| :--- | :--- |
| `--items '<JSON配列>'` | 分解済みナレッジを JSON 配列で渡す（推奨） |
| `--text "テキスト"` | 単一テキストを自動分解して保存（簡易） |
| `--file /path/to/file` | ファイルから読み込んで保存 |
| `--tags "tag1,tag2"` | タグをカンマ区切りで付与（`--text` / `--file` 時） |
| `--source "ラベル"` | 知識の出自ラベル |
| `--dedup` | 保存前に重複チェックを行う（推奨）。閾値以上の類似エントリが存在する場合はスキップして警告を表示 |
| `--dedup-threshold 0.90` | 重複判定の cosine 類似度閾値（デフォルト: `0.90`）。値を下げると重複とみなす範囲が広がる |

**`--dedup` 実行時のターミナル出力例（1件保存 + 1件スキップの場合）**:
```
Saved 1 knowledge unit(s):
  [1] <uuid>: ChromaDB は cosine 距離をデフォルトで使用する...

Skipped 1 duplicate(s):
 SKIP (sim=0.9312): ChromaDB のデフォルトポートは 8000...
```

```bash
# 分解済み JSON で保存（重複チェックあり）
python skills/long-term-memory/scripts/memory_save.py \
  --items '[
    {"text": "ChromaDB のデフォルトポートは 8000", "tags": ["chromadb", "config"]},
    {"text": "ChromaDB は cosine 距離をデフォルトで使用する", "tags": ["chromadb", "embedding"]}
  ]' \
  --source "ユーザー入力" \
  --dedup
```

### search — ナレッジの検索

| 引数 | 説明 |
| :--- | :--- |
| `--query "クエリ文"` | 自然言語クエリでベクトル類似検索 |
| `--n-results N` | 取得件数（デフォルト: `5`） |
| `--tags "tag1,tag2"` | タグでフィルタリング |
| `--json` | JSON 形式で出力（プログラム処理向け） |
| `--count` | 全保存件数を表示 |
| `--list-tags` | タグ一覧と使用頻度を表示 |

```bash
# 基本検索
python skills/long-term-memory/scripts/memory_search.py \
  --query "ChromaDB の設定方法" --n-results 5

# タグフィルタ付き検索
python skills/long-term-memory/scripts/memory_search.py \
  --query "デプロイ手順" --tags "deployment" --n-results 10
```

**検索スコアの解釈（cosine similarity）**

| スコア範囲 | 解釈 | 推奨アクション |
| :--- | :--- | :--- |
| ≥ 0.85 | 非常に高い一致（ほぼ同一） | そのまま回答に使用 |
| 0.60 - 0.84 | 関連性が高い | 回答に活用、必要に応じて補足 |
| 0.35 - 0.59 | 部分的に関連 | 参考程度、ユーザーに確認推奨 |
| < 0.35 | 関連性が低い | 無視して良い |

### update — ナレッジの更新

対象 ID は事前に `memory_search.py` で特定してください。

| 引数 | 説明 |
| :--- | :--- |
| `--id "uuid"` | 更新対象の ID（必須） |
| `--text "新テキスト"` | テキストを更新 |
| `--tags "tag1,tag2"` | タグを更新 |
| `--source "ラベル"` | ソースラベルを更新 |

```bash
python skills/long-term-memory/scripts/memory_update.py \
  --id "uuid-of-memory" \
  --text "更新後のテキスト" \
  --tags "tag1,tag2"
```

### delete — ナレッジの削除

| 引数 | 説明 |
| :--- | :--- |
| `--id "uuid"` | 単一削除 |
| `--ids "uuid-1,uuid-2"` | 複数削除（カンマ区切り） |
| `--tag "タグ名"` | タグ指定の一括削除（確認のみ） |
| `--tag "タグ名" --confirm` | タグ指定の一括削除（実行） |

```bash
# 単一削除
python skills/long-term-memory/scripts/memory_delete.py --id "uuid-of-memory"

# タグ一括削除（2 ステップ）
python skills/long-term-memory/scripts/memory_delete.py --tag "obsolete"
python skills/long-term-memory/scripts/memory_delete.py --tag "obsolete" --confirm
```

---

## タグ分類体系

一貫したタグ付けにより検索精度が向上します。

| カテゴリ | タグ例 | 用途 |
| :--- | :--- | :--- |
| ドメイン | `java`, `python`, `docker`, `k8s` | 技術領域 |
| タスク | `config`, `debug`, `deploy`, `test` | 作業種別 |
| ソース | `user-pref`, `project-rule`, `lesson` | 知識の出自 |
| プロジェクト | `bridge`, `my-copilot` | 対象プロジェクト |
| 種別 | `fact`, `rule`, `pattern`, `procedure` | 知識の性質 |

**タグ命名規則**

- 小文字のみ使用（英数字とハイフンのみ）
- 技術用語は英語で統一
- 1 エントリあたり 2〜3 個を目安（多すぎると検索ノイズになる）

---

## ファイル構成

```
memoir/
├── README.md                              # このファイル
├── apm.yml                                # APM プロジェクト設定
├── apm.lock.yaml                          # 依存バージョンロックファイル
├── plugin.json                            # プラグインメタデータ
└── skills/
    └── long-term-memory/
        ├── SKILL.md                       # スキル定義・操作ガイド
        ├── docker/
        │   └── docker-compose.yml         # ChromaDB コンテナ設定
        └── scripts/
            ├── _common.py                 # Docker 管理・ChromaDB クライアント共通処理
            ├── memory_save.py             # ナレッジ保存スクリプト
            ├── memory_search.py           # ナレッジ検索スクリプト
            ├── memory_update.py           # ナレッジ更新スクリプト
            └── memory_delete.py           # ナレッジ削除スクリプト
```
