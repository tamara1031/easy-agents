# エージェント構成

## エージェント一覧

| エージェント | モジュール | user-invocable | モデル | ツール |
|---|---|---|---|---|
| easy-agent | easy-agent | ✅ | Sonnet | read, edit, search, execute, agent, todo |
| advisor | advisor | ❌ | **Opus 4.7** | read, search |
| parliament-chairperson | parliament | ❌ | Sonnet | read, search, agent |
| parliament-member | parliament | ❌ | Sonnet | read, edit, search, execute, agent |
| hierarchy-manager | taskforce | ❌ | Sonnet | read, search, agent, todo |
| hierarchy-member | taskforce | ❌ | Sonnet | read, edit, search, execute, agent |

---

## easy-agent

**パス**: `easy-agent/agents/easy-agent.agent.md`

**役割**: ユニバーサルオーケストレーター。全ユーザーリクエストの単一エントリーポイント。

### 3軸タスク分類

| 軸 | 値 | 判定基準 |
|---|---|---|
| AmbiguityLevel | HIGH / LOW | 目標が未定義、中間状態が不明確、承認基準が不確か |
| TaskScale | Small / Mid / Large | ファイル数: 1 / 1-3 / 4以上 |
| TaskType | research / execute / hybrid / designExecute | タスクの性質 |

### Phase Pipeline

```
Explore → Deliberate → Plan → Implement → Verify → Synthesize
```

| フェーズ | ツール | 委譲先 |
|---|---|---|
| Explore | read, search | — |
| Deliberate | agent | parliament-chairperson |
| Plan | todo | hierarchy-manager (Large) |
| Implement | edit, execute | hierarchy-manager (4ファイル以上) |
| Verify | execute | — |
| Synthesize | — | — |

### Phase Gate Protocol

| ステータス | 意味 |
|---|---|
| APPROVED | 次フェーズへ進む |
| REVISE | 同フェーズを修正して再実行 |
| DELEGATE | 専門エージェントへ委譲 |
| LOOPBACK | 前フェーズに戻る |
| ESCALATE | エスカレーション (要ユーザー確認) |

### フォールバックチェーン

- Verify 失敗 → Implement 再試行 (最大5回)
- Deliberate 停滞 → Explore 再試行
- Plan 破綻 → Deliberate または Explore 再試行

---

## advisor

**パス**: `advisor/agents/advisor.agent.md`

**役割**: 戦略的アドバイザー。Opusモデルで高品質な判断を提供。実行は行わない。

### 相談トリガー (7種)

| # | トリガー | タイミング |
|---|---|---|
| 1 | 目標未定義 | 実行前 |
| 2 | 中間状態不明 | 実行前 |
| 3 | 実現可能性不明 | 実行前 |
| 4 | スコープ肥大化 (3回以上試行して詰まった) | 実行中 |
| 5 | 不可逆性の懸念 | 実行前 |
| 6 | 完了後の残留リスク | 実行後 |
| 7 | フェーズゲート遷移 | フェーズ移行時 |

### 返答フォーマット (XML)

```xml
<analysis>状況分析 (1-3文)</analysis>
<recommended_approach>番号付きステップ</recommended_approach>
<risks>懸念点または「なし」</risks>
<verdict>PROCEED / CORRECT / ESCALATE / STOP</verdict>
<!-- ESCALATE の場合のみ -->
<escalation_target>hierarchy / parliament</escalation_target>
<escalation_reason>理由 (1-2文)</escalation_reason>
```

---

## parliament-chairperson

**パス**: `parliament/agents/parliament-chairperson.agent.md`

**役割**: 議会議長。トピックを分析し、メンバーペルソナを生成、討議を進行してコンセンサスを形成。

### 必須ペルソナ (4名)

| ペルソナ | 役割 |
|---|---|
| Advocate (推進者) | アイデア推進・創造的提案 |
| Reviewer/Critic (批判者) | 欠陥・リスク指摘 |
| Compliance (法規制/倫理) | 法的・倫理・セキュリティ審査 |
| Pragmatist (現実主義者) | コスト・工期・実現可能性管理 |

オプション追加ペルソナ: Domain Expert, User Advocate, Performance Engineer, Security Specialist

### 収束条件

| 条件 | 説明 |
|---|---|
| AGREED | 全メンバーが承認または軽微な修正のみ |
| CONVERGED | `convergence_threshold` 回連続で新論点なし |
| MAX_ROUNDS | `max_rounds` 到達で強制終了 |

---

## parliament-member

**パス**: `parliament/agents/parliament-member.agent.md`

**役割**: 議員。割り当てられたペルソナの視点で批判的にレビューし、合意形成に参加。

### スタンス

| スタンス | 意味 | condition_for_approval |
|---|---|---|
| PROPOSE | 新しい提案 | null |
| CRITIQUE | 問題・リスク指摘 | **必須** |
| APPROVE | 現状提案を承認 | null |
| REVISE | 軽微な修正提案 | null |

---

## hierarchy-manager

**パス**: `taskforce/agents/hierarchy-manager.agent.md`

**役割**: 階層型マネージャー。大規模タスクを Plan→Implement→Review サイクルで管理。

### 生成ペルソナ

| ペルソナ | 役割 | 必須/任意 |
|---|---|---|
| Planner | 詳細計画作成 | 必須 |
| Implementer | 成果物作成 | 必須 |
| Reviewer | 品質検証 | 必須 |
| Domain Expert / Security / Performance / Test / UX | 専門視点 | 任意 (最大2名追加) |

### 内部フィードバックループ

Implementer → Reviewer → (REVISE の場合) → Implementer (最大5回)

---

## hierarchy-member

**パス**: `taskforce/agents/hierarchy-member.agent.md`

**役割**: 階層型サブエージェント。Manager から指示を受け、実際の実装・検証を実行。

### ロール別品質ガード

**Implementer**:
- テスト専用ハック (`if (isTest)` 分岐) 禁止
- マジックナンバー・ハードコーディング禁止
- チェックリスト要件のみ実装 (YAGNI)

**Reviewer**:
- テストハック検出
- API/パターンの実在確認
- 要件充足検証
