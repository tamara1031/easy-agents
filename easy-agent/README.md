# easy-agent

Claude Code 向けの汎用エージェントです。ユーザーの要求を受け取ると、その複雑さと種別を自動判定し、適切な実行フローを組み立てて自律的にタスクを完遂します。小さなバグ修正から大規模なアーキテクチャ設計まで、単一のエントリーポイントで扱えます。

## サブエージェント委譲の判断基準

| TaskType | TaskScale | 委譲先 |
| :--- | :--- | :--- |
| `research` | 任意 | なし（easy-agent が直接実行） |
| `execute` / `hybrid` | Small | なし（easy-agent が直接実行） |
| `execute` / `hybrid` | Mid / Large | **Hierarchy** (taskforce) |
| `designExecute` | 任意 | **Parliament** (設計合意) → その後 **Hierarchy** (実装) |
| 任意 | 任意、判断が困難 | **Advisor** (on-demand、最大3回) |

## 主な機能

- **3軸タスク分類 (曖昧度 × TaskScale × TaskType)**: 要求を自動分析し、`research` / `execute` / `hybrid` / `designExecute` の4種のフェーズパイプラインへ振り分ける
- **Phase Gate Protocol**: 各フェーズ完了時に品質評価（APPROVED / REVISE / DELEGATE / LOOPBACK / ESCALATE）を実施し、ループバックや委譲を動的に制御する
- **On-demand Advisory**: フェーズ実行中に複雑な設計判断やトレードオフが生じた際、`advisor` サブエージェントへ随時相談できる（1タスクあたり最大3回）
- **階層的委譲**: `TaskScale = Mid/Large` でサブエージェントへ委譲する（判断基準: `execute`/`hybrid` → Hierarchy、`designExecute` → Parliament）
- **前処理ガード**: 不可逆操作・高コスト処理を実行前に検知し、ユーザー確認ゲートを強制発動する

## 前提条件

| 項目 | 要件 |
| :--- | :--- |
| パッケージバージョン | 0.1.0 |
| 利用ツール | `read`, `edit`, `search`, `execute`, `agent`, `todo` |
| 依存サブエージェント | `hierarchy-manager`, `parliament-chairperson`, `advisor` |
| APM 宣言依存 | `advisor`, `parliament`, `taskforce`, `memoir`, `refine-loop`（`apm.yml` の `dependencies.apm`） |

## インストール方法

APM を使用してインストールします。

```bash
apm install easy-agent
```

または、リポジトリを直接クローンして利用します。

```bash
git clone https://github.com/easy-agents/easy-agent
cd easy-agent
```

`apm.yml` で `advisor` / `parliament` / `taskforce` / `memoir` / `refine-loop` を宣言依存しているため、`apm install` 時に自動的に解決・インストールされます。

## 基本的な使い方

easy-agent はユーザーの要求を受け取ると、まず以下のタスク分析を自動実行してユーザーに提示します。

```markdown
# タスク分析結果報告
* **AmbiguityLevel**: LOW
* **TaskScale**: Small（変更ファイル: src/auth.ts のみ）
* **TaskType**: hybrid（調査 + 修正）
* **Phase Pipeline**: Explore → Plan → Implement → Verify
* **残存リスク**: なし
* **推定コスト**: Low

## 実行計画
1. [Explore] src/auth.ts の null チェック漏れ箇所を特定
2. [Plan] 修正方針を決定（Optional チェーン / 防御的代入）
3. [Implement] 修正を適用
4. [Verify] 既存テストが通ることを確認

続行しますか？ [ yes / no / スコープを変更 ]
```

### エンド・ツー・エンド例: バグ修正

```
ユーザー: "src/auth.ts の null pointer exception を直して"

→ 分類: TaskType=hybrid, TaskScale=Small, Pipeline: Explore→Plan→Implement→Verify
→ [Explore] auth.ts を読み込み、変数 user が null になり得る行を特定
→ [Plan] optional チェーン（user?.id）で修正する方針を決定
→ [Implement] auth.ts の該当行を修正
→ [Verify] npx jest src/auth.test.ts を実行 → PASS
→ 完了報告: "src/auth.ts 42行目のnullチェック漏れを修正しました。テスト全件PASS。"
```

### タスク分類の例

| ユーザーの要求例 | TaskType | Phase Pipeline |
| :--- | :--- | :--- |
| 「このライブラリの仕様を調べてまとめて」 | `research` | Explore → Synthesize |
| 「変数名を修正して」（明確・小規模） | `execute` | Plan → Implement → Verify |
| 「このバグを調査して修正して」 | `hybrid` | Explore → Plan → Implement → Verify |
| 「認証モジュールのアーキテクチャを設計して実装して」 | `designExecute` | Explore → Deliberate → Plan → Implement → Verify |

### TaskScale の基準

| ラベル | 変更ファイル数 | 例 |
| :--- | :--- | :--- |
| `Small` | 1ファイルのみ | 変数リネーム、ドキュメント修正 |
| `Mid` | 2〜3ファイル | 単一機能の追加（実装＋テスト） |
| `Large` | 4ファイル以上 | アーキテクチャ変更、大規模リファクタリング |

### AmbiguityLevel の基準

| レベル | 基準 |
| :--- | :--- |
| `HIGH` | ゴール・成果物・変更範囲のいずれかが不明確。Explore フェーズを先行させ、完了後に TaskScale と TaskType を再判定する |
| `LOW` | ゴール・成果物・変更範囲がすべて明確。即座に Plan or Implement から開始できる |

### execute / hybrid / designExecute の判別

| TaskType | 判定基準 |
| :--- | :--- |
| `execute` | 何をすべきかが明確で、調査や設計判断が不要 |
| `hybrid` | 実装前に既存コードの調査が必要（バグ調査、仕様確認） |
| `designExecute` | 「どう設計するか」という判断が必要（アーキテクチャ選定、技術選定、システム構造の変更）。リファクタリングでも既存構造を踏襲するなら `execute`、新たな設計判断を含む場合は `designExecute` |


## 定期バッチ運用時のブランチ同期チェック

`easy-agent` を定期実行ジョブに組み込む場合、実行前に次の順序で作業ブランチの健全性を確認します。

1. `git fetch origin main`
2. `git merge origin/main`
3. `apm install`（`apm.yml` や `apm.lock.yaml` が更新されていた場合のみ）
4. `git diff --check`（競合マーカーや空白エラー検出）

`origin` が未設定のローカル検証環境では、まず `git remote -v` で接続先を確認し、必要に応じて `git remote add origin <url>` を行ってから同期してください。

## ファイル構成

```
easy-agent/
├── agents/
│   └── easy-agent.agent.md   # エージェント本体の定義。3軸分類・フェーズパイプライン・Phase Gate Protocol・委譲戦略などの全ルールを記述
├── apm.yml                   # APM パッケージ設定（依存: advisor, parliament, taskforce, memoir）
├── plugin.json               # プラグインメタデータ（名前・バージョン・説明）
├── README.md                 # 本ファイル
└── .gitignore                # APM コンパイル成果物の除外設定
```

> APM 依存関係のロックは、ルートワークスペースの `apm.lock.yaml`（リポジトリ直下）で集中管理されます。サブパッケージごとのロックファイルは存在しません。

### 主要ファイルの詳細

**`agents/easy-agent.agent.md`**
エージェントの全動作ロジックを定義するコアファイルです。以下のセクションで構成されています。

- `3-axis Classification`: 曖昧度・TaskScale・TaskType の判定ルール
- `Phase Pipeline`: 各フェーズ（Explore / Deliberate / Plan / Implement / Verify / Synthesize）の目的とツール
- `Phase Gate Protocol`: フェーズ遷移の評価基準と Advisory トリガー条件
- `Escalation Criteria`: Hierarchy / Parliament へのエスカレーション判断基準
- `Pre-processing Guards`: 不可逆性ガード・コストガード
- `Constraints`: 確認ゲートのスキップ禁止など8つの制約ルール

**`apm.yml`**
APM プロジェクトの設定ファイルです。`dependencies.apm` で `advisor` / `parliament` / `taskforce` / `memoir` を宣言依存し、`apm install` 時に同時インストールされます。
