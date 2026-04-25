# taskforce

大規模タスクを **Plan→Implement→Review** サイクルで自律的に完遂する階層型エージェントパッケージ。Orchestrator が複数の Manager を並列ディスパッチし、各 Manager が Planner・Implementer・Reviewer のロールを持つ Member を生成して品質ゲートを通過するまで反復する。

---

## Generator-Verifier パターン

`call-hierarchy` の内部は **Generator-Verifier** の構造で動作する。

| 役割 | エージェント | 責務 |
| :--- | :--- | :--- |
| **Generator** | Implementer (Member) | チェックリストを満たす成果物を生成する。品質の一次責任を担う |
| **Verifier** | Reviewer (Member) | チェックリストの各項目に `PASS`/`FAIL` を明示判定し、`FAIL` があれば具体的な修正指示 (`rejection_instructions`) を Generator に返す |

Reviewer は「良い/悪い」の主観評価ではなく、チェックリスト項目への客観的な合否判定のみを行う。`REVISE` が返された場合、Manager は `rejection_instructions` を次の Implementer の `rejection_reason` に設定して再実行し、`APPROVE` が得られるまで内部ループを継続する（最大 5 回）。

---

## タスク分解の原則：コンテキスト中心の分離

タスクは「作業領域」ではなく「必要なコンテキスト」で分割する。

| 良い分割 | 悪い分割 |
| :--- | :--- |
| 「認証モジュールの実装」（同一ファイル群、対照が容易） | 「コードを書く」「テストを書く」（同じコンテキストが必要） |
| 「API エンドポイント A」と「API エンドポイント B」（独立） | 「ルーティング」と「ビジネスロジック」（密結合） |
| 最小スコープで閲覧範囲が網羅できるタスク | 依存関係が複雑で他タスクの完了待ちが頻発するタスク |

各タスクに `depends_on`（前提タスク ID リスト）を定義し、完了次第後続タスクへ出力 (`output`) を連鎖させるローリング方式で実行する。

**`depends_on` の定義例（Orchestrator が Phase 1 でタスク一覧を作成する際に設定）**

```
T001: DBマイグレーション実行       depends_on: []          → 最初に実行
T002: Repositoryクラス実装        depends_on: ["T001"]    → T001 完了後に開始
T003: Serviceクラス実装           depends_on: ["T002"]    → T002 完了後に開始
T004: APIコントローラー実装        depends_on: ["T003"]    → T003 完了後に開始
T005: テスト追加                  depends_on: ["T004"]    → T004 完了後に開始
```

前段タスクが `APPROVED` になると、そのタスクの `output`（Manager が報告する実装サマリー）が後続タスクの `{context}` フィールドへ自動注入されます。

---

## いつ使うか / 使わないか

### 使うべきケース

- 変更対象ファイルが 3 つ以上にわたり、独立した単位に分解できる
- 並列実装・役割分担が有効な中〜大規模タスク
- 「計画→実装→レビュー」の明確なサイクルが品質向上に寄与する作業
- ユーザーが「並列で実装して」「役割分担して進めて」と指示した場合

### 使わないケース

| 状況 | 理由 |
| :--- | :--- |
| 変更対象が 1〜2 ファイル | サブエージェントのオーバーヘッドが成果を上回る |
| タスクが 15 分以内に完了する見込み | 起動コスト・調査コストが割に合わない |
| 目的が明快な小タスク | 単純タスクへの階層適用は過剰設計 |
| 戦略的な設計変更（design-Execute） | `Parliament`（複数エージェント会議）への委譲を優先する |

---

## 主要パラメータ一覧

| パラメータ | 説明 | デフォルト値 |
| :--- | :--- | :--- |
| `parallelism` | Manager の同時起動スロット数 | `5` |
| `max_rejections` | 1 タスクあたりの差し戻し上限回数（超過でユーザーエスカレーション） | `3` |
| `member_count` | Manager が生成する Member 数（必須ロール 3 名 + 追加最大 2 名） | `3` |
| `checklist_path` | チェックリストファイルの保存パス | `docs/hierarchy-checklist.md` |

### Manager 内部ループ上限（固定値）

| 項目 | 上限 |
| :--- | :--- |
| Implementer→Reviewer の内部フィードバックループ | 5 回 |
| 3 回目の REVISE で同一項目が再 FAIL → 根本解決失敗として扱う | — |

---

## タスクステータス遷移図

```
TODO
  └─(Manager 起動)──→ IN_PROGRESS
                          └─(Manager 完了)──→ IN_REVIEW
                                                ├─(Orchestrator APPROVED)──→ APPROVED  ← ゴール
                                                └─(Orchestrator REJECTED)──→ REJECTED
                                                                              └─(再キュー)──→ TODO

任意のステータス ─(Manager 異常終了)──→ ERROR
                                          └─(ユーザー「再試」)──→ TODO
```

### ERROR 発生時の回復手順

1. Orchestrator がエラー内容（エラーメッセージ・失敗した checklist 項目）をユーザーに報告する
2. 根本原因を特定する（コード依存関係の問題 / checklist 条件が実現不可能 / 外部サービス障害 など）
3. 原因を修正する（コード修正 / checklist 条件の見直し / 環境設定の修正）
4. ユーザーが「T00X を再試して」と指示 → Orchestrator が該当タスクを `TODO` に戻して再キューに追加

> 再試行時は Manager プロンプトの `# 差し戻し理由` フィールドに障害の概要を記入すると、次の Planner が経緯を把握した上で計画を立て直せます。
> **例 (ERROR 起因の再試行)**: `"T003 異常終了: T002 の output パスが不一致 (expected: src/repo.ts, actual: src/repository.ts)。パス修正後に再試行。"`

### ステータス更新権限

| 遷移先 | 更新権限 |
| :--- | :--- |
| `TODO` | Orchestrator（初期化時・差し戻し再キューイング時） |
| `IN_PROGRESS` | Orchestrator のみ（Manager ディスパッチ時） |
| `IN_REVIEW` | Manager のみ（作業完了時、`manager_output.json` で報告） |
| `APPROVED` / `REJECTED` / `ERROR` | Orchestrator のみ |

> Manager が `APPROVED`/`REJECTED`/`ERROR` へ遷移させることは禁止。

---

## 使い方

### 前提条件

VS Code 環境で `runSubagent` を使用する場合：

```json
"chat.subagents.allowInvocationsFromSubagents": true
```

CLI 環境で `task` ツールを使用する場合、上記設定は不要。

### Orchestrator から `call-hierarchy` を起動する

#### CLI パターン（`task` ツール）

```bash
task(
  agent_type: "my-copilot:hierarchy-manager",
  mode: "background",
  name: "manager-T001",
  description: "認証モジュールの実装 (短期版)",
  prompt: """
# タスクの概要
T001
JWT を使用したユーザー認証モジュールを実装する

# チェックリストファイル
docs/hierarchy-checklist.md

# 承認条件チェックリスト
- [ ] UserService.authenticate() が JWT を検証して User を返す
- [ ] 無効トークン時に AuthException をスローする
- [ ] 単体テストが全件 PASS する

# メンバー数
3

# 差し戻し理由 (再試行の場合)


# 補足コンテキスト

  """
)
```

#### VS Code パターン（`runSubagent`）

```javascript
runSubagent(
  agentName: "hierarchy-manager",
  description: "認証モジュールの実装 (短期版)",
  prompt: "..."  // 上記と同じテンプレートを使用
)
```

### タスク分割計画の承認フロー

Orchestrator は Manager 起動前にユーザーへ以下のフォーマットで承認を求める（承諾なしの起動は禁止）。

```
タスク分割計画の承認をお願いします
[議題] {大まかな目標1行}

| ID   | タスク内容        | 担当ロール      | 承認条件数 |
| :--- | :---             | :---           | :---      |
| T001 | 認証モジュール実装 | 3名(P/I/R)     | 3         |
| T002 | APIエンドポイント  | TBD(内部で決定) | 2         |

パラメータ: parallelism(5), max_rejections(3), member_count(3)
この計画で進めてよろしいですか？
```

### 進捗レポート形式

Orchestrator はスロット補充のたびにコンパクトな進捗を報告する。

```
| APPROVED:    | 2件 ( T001, T002 )                    |
| REJECTED:    | 1件 ( T003 | チェックリスト項目2 FAIL ) |
| IN_PROGRESS: | 2件 ( 実行中... )                     |
| 残キュー:    | 3件                                   |
```

---

## ファイル構成

```
taskforce/
├── plugin.json                                # パッケージメタ情報
├── agents/
│   ├── hierarchy-manager.agent.md             # Manager サブエージェントテンプレート
│   └── hierarchy-member.agent.md              # Member サブエージェントテンプレート
└── skills/
    └── call-hierarchy/
        ├── SKILL.md                           # スキル定義・ワークフロー・パラメータ仕様
        ├── schemas/
        │   ├── manager_output.json            # Manager→Orchestrator 出力スキーマ
        │   ├── member_output.json             # Member→Manager 出力スキーマ
        │   └── orchestrator_state.json        # Orchestrator 状態管理スキーマ
        └── templates/
            ├── checklist.md                   # チェックリストテンプレート
            └── status_definitions.md          # ステータス定義・遷移規則
```
