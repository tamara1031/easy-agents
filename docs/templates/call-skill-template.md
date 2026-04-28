<!--
新規 call-X スキル作成用のテンプレート。
使い方:
  1. このファイルを `<package>/.apm/skills/call-<skill-name>/SKILL.md` にコピー
  2. {placeholder} と {# コメント} を実際の内容に置換
  3. 不要な節（オプション扱い）は削除; 必須節（★）は削除禁止
  4. CI (`.github/workflows/check.yml`) が以下を機械チェックする:
     - ADR-009: Caller Response Contract セクションの存在
     - ADR-013: Context Window Management セクションの存在
  5. `docs/skills.md` の一覧表に行を追加し、必要なら `docs/agents.md` にも対応エージェントを追加
詳細: docs/README.md の「新規 call-X スキルの追加」を参照
-->

---
name: call-{skill-name}
description: "{1-2 文。スキルが何を行い、どんな状況で起動されるか。例: 'call-X (XX パターン) を起動するスキル。…する場合に発動する。'}"
user-invocable: false
tools: [read, edit, search, execute, agent]
agents: [{agent-name}]
---

## Concept

{2-3 段落。このスキルが体現する抽象パターン (Advisor / Generator-Verifier / 議会モデル / 反復改善等) と、
中核的な前提・仕組みを述べる。}

## When to use

- {使うべき場面 1}
- {使うべき場面 2}

## When NOT to use

| 状況 | 代替 |
| :--- | :--- |
| {状況} | {代替手段} |

## How To Call

利用可能なツールに応じて呼び出し方を切り替える。

### Claude Code パターン (`agent` ツール)

```
agent(
  subagent_type: "{agent-name}",
  description: "{short description}",
  prompt: "<下記テンプレートに従って構築>"
)
```

### CLI パターン (`task` ツール利用時)

```bash
task(
  agent_type: "my-copilot:{agent-name}",
  mode: "background",
  name: "{task_id}",
  description: "{short description}",
  prompt: "<下記テンプレートに従って構築>"
)
```

### VS Code パターン (`runSubagent` ツール利用時)

```javascript
runSubagent(
  agentName: "{agent-name}",
  description: "{short description}",
  prompt: "<下記テンプレートに従って構築>"
)
```

## Parameters

| パラメータ | 説明 | デフォルト値 |
| :--- | :--- | :--- |
| `{name}` | {description} | `{default}` |

## Workflow

{# 単一フェーズの場合は箇条書き、複数フェーズの場合は parliament/hierarchy に倣い Phase 1/2/3/4 形式で記述する}

### Phase 1: {初期化等}

1. ...

### Phase 2: ...

---

## 呼び出し元の応答コントラクト (Caller Response Contract)

> **必須セクション**: ADR-009 規約 1 に準拠。`.github/workflows/check.yml` の `Lint Caller Response Contract`
> ステップで本見出しの存在が機械チェックされる。

call-{skill-name} を呼び出したエージェント（通常 easy-agent の {phase} フェーズ）が、各返却ステータスを受け取った際に取るべきアクションを定義する。

{# パターン A: 単一階層 (advisor / refine-loop と同型 — サブエージェントが処理単位を持たない場合)
   下記の表を採用し、後続の「2層構造」セクションは削除する。}

| ステータス | 意味 | 呼び出し元が取るべきアクション |
| :--- | :--- | :--- |
| `{STATUS_OK}` | {成功時の意味} | {次フェーズへ進める / リソース解放 / etc.} |
| `{STATUS_PARTIAL}` | {部分的成功・時間切れ等} | {ユーザーに残存課題を提示し選択を仰ぐ等} |
| `{STATUS_ESCALATE}` | {根本問題・委譲必要} | {上位フェーズに差し戻す / Advisory 相談へ等} |
| `{STATUS_ABORT}` | {実行不能} | {STOP / 環境チェック後再試行等} |
| `DISPATCH_FAILURE` | サブエージェントの起動失敗・タイムアウト・ツール不可（[ADR-015](../adr/ADR-015-dispatch-failure-protocol.md)） | {スキルの分類に応じてフォールバック戦略を選択: Degrade-and-Continue (advisor型) / Skip-and-Report (parliament・hierarchy型) / Fallback-Mode (refine-loop型)} |

{# パターン B: 2層構造 (parliament / hierarchy と同型 — サブエージェントが議題/タスク等の複数処理単位を持つ場合)
   上記の単一階層表を削除し、以下の2表を採用する。}

### 単位 (per-unit) の返却ステータス

| ステータス | 意味 | 呼び出し元が取るべきアクション |
| :--- | :--- | :--- |
| `{PER_UNIT_OK}` | {単位ごとの成功} | {当該単位を APPROVED に遷移} |

### オーケストレーター集約レベルの返却ステータス

| ステータス | 根本原因 | 呼び出し元が取るべきアクション |
| :--- | :--- | :--- |
| 全単位 APPROVED | {全成功} | {次フェーズへ進む} |
| `max_rejections` 超過 | {差し戻し上限超過} | {サブエージェントが提示した選択肢をそのままユーザーへ転送する (Relay Principle)} |
| `DISPATCH_FAILURE` | サブエージェントの起動失敗・タイムアウト・ツール不可（[ADR-015](../adr/ADR-015-dispatch-failure-protocol.md)） | **Skip-and-Report**: 失敗した処理単位を ERROR 扱いでスキップし、残存単位を継続する。全単位失敗時は Phase Gate で STOP |

> **{紛らわしいステータス対}の違い**: {境界条件を1-2文で明記}

> **Source of Truth**: 呼び出し元の高位エージェント（easy-agent 等）は本表を Source of Truth として参照する。
> `easy-agent.agent.md` の Fallback Chain と本表がドリフトした場合、本表が正となる（[ADR-009](../../../docs/adr/ADR-009-caller-response-contract-convention.md) 規約 4）。

---

## Constraints

1. {制約 1: 例 — オーケストレーターは議論に参加しない}
2. {制約 2}

## Verification Criteria (検証基準) {# オプション}

- {検証項目 1}

## Context Window Management (コンテキスト管理)

> **必須セクション**: ADR-013 に準拠。`.github/workflows/check.yml` の `Lint Context Window Management declaration`
> ステップで本見出しの存在が機械チェックされる。

### {エージェント}への委譲時

{# 呼び出し先エージェントへのコンテキスト最小化方針を記述する。例:}

1. **コンテキストの最小化**: {エージェント}プロンプトには{スキル固有の情報}のみを含める。
2. **チェックリストは完全に渡す**: {エージェント}が正確に判断できるよう、チェックリストは省略しない。
3. **補足コンテキストの要約**: `{context}` は関連コードの抜粋 (5-20行) と要約のみ。上限500トークン。

### コンテキスト爆発防止

{# 多段起動・ループ・並列実行が発生する場合の爆発防止策を記述する。}

1. **{防止策 1}**: {説明}
2. **{防止策 2}**: {説明}

### トークン予算

{# ADR-013 §2 のフォーマットに従い、呼び出し階層ごとに入力上限・出力上限を記述する。
   階層 = 「呼び出し元 → 呼び出し先」のペア。}

| 階層 | 入力上限 | 出力上限 |
| :--- | :--- | :--- |
| オーケストレーター → {エージェント} | 1,000トークン ({context}: 500, {checklist}: 300, {topic/task}: 200) | — |
| {エージェント} → {サブエージェント} (各{ラウンド/イテレーション}) | 700トークン ({内訳}) | 500トークン ({内訳}) |
| {エージェント} → オーケストレーター | — | 600トークン ({出力形式}) |

> **超過時の対応**: 補足コンテキスト (`context`) を段階的に削減: 500 → 300 → 100トークン。
> それでも超過する場合は `context` を「チェックリスト達成に直結する情報のみ」に絞り込む。
> {スキル固有の追加ルール}
