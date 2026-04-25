# Parliament パッケージ

複数の専門家エージェントによる多視点議論を通じて、複雑な設計・方針の判断を合意形成する「議会モデル」の実装パッケージです。単一エージェントでは偏りが生じやすい判断を、対立役割を持つ議員の構造化された議論によって質の高い合意へ収束させます。

---

## 仕組み

議会モデルは **3層の役割** で構成されます。

```
ユーザー
  └─ オーケストレーター (現在の実行エージェント)
       ├─ parliament-chairperson (議長) — 議題ごとに1体
       │    ├─ parliament-member: Advocate
       │    ├─ parliament-member: Reviewer
       │    ├─ parliament-member: Compliance
       │    └─ parliament-member: Pragmatist  [+ 追加ロール]
       └─ parliament-chairperson (議長) — 別議題 (並列)
            └─ ...
```

| 役割 | 責務 |
| :--- | :--- |
| **オーケストレーター** | 大目標を議題に分割し、承認条件チェックリストを作成。議長を起動して成果物を検収・統合する。**自ら議論には参加しない。** |
| **Chairperson（議長）** | 議題ごとに1体起動。議員ペルソナを動的生成し、議論をファシリテート。収束判定・要約・成果物のオーケストレーターへの提出を担う。**意見を述べない。** |
| **Member（議員）** | 議長が生成。専門の観点から発言し、チェックリストに対する承認/修正/批判のスタンスを JSON で返答する。 |

---

## 必須ペルソナ 4 名

議長はすべての議題で以下の 4 役割を必ず配置します。

| ロール | 目的 |
| :--- | :--- |
| **Advocate（推進者）** | 議題を前進させるアイデア出しと創造的な提案。競合案が複数ある場合は各案に 1 名ずつ配置する。 |
| **Reviewer（批判者）** | 提案の欠陥・品質リスクを鋭く指摘し、全体の品質を担保する。 |
| **Compliance（倫理/法規制）** | 法規制・セキュリティ・倫理観点の審査を行う。 |
| **Pragmatist（現実主義者）** | コスト・納期・実現可能性を管理し、実行可能な落としどころを探る。 |

`member_count` が 5 以上の場合、議長は追加ロール（Domain Expert / User Advocate / Performance Engineer / Security Specialist など）を動的に生成できます。

---

## いつ使うか / 使わないか

### 使うべき場面

- ユーザーが「議論して」「多角的に検討して」「設計をレビューして」「方針を決めて」と言った場合
- 複数の競合アプローチがありトレードオフの評価が必要な場合
- アーキテクチャ・技術選定など、後から覆しにくい重要な判断
- 複数ドメインにまたがる複雑な設計判断

### 使わない場面

| 状況 | 代替手段 |
| :--- | :--- |
| 仕様が確定済みで設計判断が不要 | Hierarchy で直接実装 |
| 明確なベストプラクティスが存在する | 直接実行 |
| 1 つの視点だけで判断可能 | Advisory で確認後に実行 |
| ユーザーが既に方針を明示している | 方針に従って Hierarchy で実装 |

---

## 主要パラメータ一覧

| パラメータ | 説明 | デフォルト値 |
| :--- | :--- | :---: |
| `parallelism` | 議題の同時並列実行最大数 | `3` |
| `member_count` | 議論に参加するメンバー数（最低 4）。競合案が複数ある場合は **競合案数 + 3** を設定（例：3 案 → 6 名）。議長は別カウント。 | `4` |
| `max_rounds` | 議論ラウンドの上限 | `5` |
| `max_rejections` | 1 議題あたりの差し戻し上限回数 | `3` |
| `summary_interval` | 議長が要約を挟む発言数間隔 | `4` |
| `convergence_threshold` | 新規論点が出ないラウンド数で早期終了する閾値 | `2` |

> **`member_count` の調整責務**: オーケストレーターが競合案数を把握している場合は事前に設定します。把握していない場合、議長が議題分析時に検出して自ら増加させます。

---

## 使い方

### 前提条件

VS Code 環境で `runSubagent` を使用する場合は、以下の設定を有効にしてください。

```json
"chat.subagents.allowInvocationsFromSubagents": true
```

CLI 環境 (`task` ツール使用時) はこの設定は不要です。

### CLI パターン (`task` ツール利用時)

`call-parliament` スキルをトリガーすると、オーケストレーターが自動的に以下のフローを実行します。

```bash
# 議長を background タスクとして起動する例
task(
  agent_type: "my-copilot:parliament-chairperson",
  mode: "background",
  name: "chair-T001",
  description: "APIスキーマ設計の方針決定",
  prompt: """
## 議題ID
T001

## 議題タイトル
APIスキーマ設計の方針決定

## 承認条件チェックリスト
- [ ] REST / GraphQL のトレードオフが明示されている
- [ ] セキュリティ要件への対応が含まれている
- [ ] 移行コストの見積もりが提示されている

## サマリー間隔
4

## メンバー数
4  # 競合案がある場合は「競合案数 + 3」に変更すること (例: 3案 → 6)

## 最大ラウンド数
5

## 収束閾値 (新規論点なしで早期終了するラウンド数)
2

## 補足コンテキスト
現在の API は REST。GraphQL 移行を検討中。
"""
)
```

- 依存関係のない議題は **同時に複数起動** します（逐次起動禁止）。
- 完了通知後、`read_agent` で `chairperson_output.json` 形式の結果を取得します。

### VS Code パターン (`runSubagent` ツール利用時)

```javascript
runSubagent(
  agentName: "parliament-chairperson",
  description: "APIスキーマ設計の方針決定",
  prompt: "... (上記テンプレートと同様の内容) ..."
)
```

### ワークフロー概要

| フェーズ | 内容 |
| :--- | :--- |
| **Phase 1: 初期化** | 大目標を議題 (`T001`, `T002`, ...) に分割し、チェックリストを作成。ユーザーの承認を得る。上流エージェントがすでにユーザー承認を取得済みの場合はスキップ可。 |
| **Phase 2: 議論の実行** | 最大 `parallelism` 件を並列起動。完了次第ローリング方式で補充。 |
| **Phase 3: 個別検収** | チェックリストを満たしているか審査。未達の場合は差し戻し（最大 `max_rejections` 回）。 |
| **Phase 4: 全体統合** | 全成果物を集約し、矛盾・トレードオフを確認後に最終レポートを生成。 |

> **`max_rejections` 超過時のフォールバック**: 差し戻し上限に達した場合、オーケストレーターは現時点の最善案を保全し、ユーザーに (a) 複数案の手動選択 / (b) 要件緩和 / (c) Advisory 相談後に再議論 の選択肢を提示します。

---

## ファイル構成

```
parliament/
├── agents/
│   ├── parliament-chairperson.agent.md   # 議長エージェントの定義テンプレート
│   └── parliament-member.agent.md        # 議員エージェントの定義テンプレート
├── skills/
│   └── call-parliament/
│       ├── SKILL.md                      # スキル本体（ワークフロー・パラメータ定義）
│       ├── schemas/
│       │   ├── chairperson_output.json   # 議長 → オーケストレーター の出力スキーマ
│       │   ├── member_message.json       # 議員 → 議長 の発言スキーマ
│       │   └── orchestrator_state.json   # オーケストレーターの状態管理スキーマ
│       └── templates/
│           └── stance_definitions.md     # スタンス定義 (PROPOSE / CRITIQUE / APPROVE / REVISE)
├── plugin.json                           # プラグインメタデータ
└── apm.yml                               # APM パッケージ設定
```

### スキーマ早見表

| ファイル | 用途 |
| :--- | :--- |
| `schemas/member_message.json` | 議員が議長へ返す JSON の必須フィールド (`agent_role`, `stance`, `target_agent`, `statement`, `condition_for_approval`) |
| `schemas/chairperson_output.json` | 議長がオーケストレーターへ返す最終成果物の構造 |
| `schemas/orchestrator_state.json` | オーケストレーターが議題の進捗・ステータスを管理する状態オブジェクト |
| `templates/stance_definitions.md` | `PROPOSE` / `CRITIQUE` / `APPROVE` / `REVISE` の定義とロール別ガイダンス |
