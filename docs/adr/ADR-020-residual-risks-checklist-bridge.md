# ADR-020: Residual Risks Checklist Bridge (Implement → Verify フェーズ間継承)

## ステータス

Accepted

## コンテキスト

ADR-018 は Hierarchy の Reviewer が non-critical FAIL を `risks` に記録し、Manager が `residual_risks` へ転記してオーケストレーターへ提出する「二段階収束モデル」を確立した。しかし ADR-018 決定文（ルール 4）では、Caller（easy-agent）が `residual_risks` を **refine-loop の `task_context` に引き継ぐ**と規定していた。

この設計には以下の情報損失が存在する:

1. **検証の非対称性**: `task_context` はレビュアーへの背景情報（上限 400トークン）であり、要件チェックリストとして正式に追跡されない。レビュアーは `residual_risks` を「知っているかもしれない」が、それを明示的に PASS/FAIL 判定する義務がない。

2. **トレーサビリティの欠如**: Verify フェーズで残存リスクが解決されたか否かが `refine_loop_output.residual_issues` に現れない。Implement フェーズの non-critical FAIL は `residual_issues` の候補となるはずだが、チェックリスト項目でなければ対象外となる。

3. **品質モデルの不完全な統一**: ADR-018 は「Implement と Verify で同一の [critical] / non-critical 二段階モデルを使う」という意図を持っていたが、Verify フェーズへの橋渡し手順が未定義だったため、統一が片面のみで完結していた。

## 決定

**Hierarchy が返した `residual_risks` は、Verify フェーズの refine-loop 呼び出し時に `requirements_checklist` の non-critical 項目（`[critical]` タグなし）として追記する。`task_context` には含めない。**

### ルール

1. **Caller（easy-agent の Implement → Verify 遷移）** は `manager_output.residual_risks` の各エントリを `requirements_checklist` にそのまま追記する。`[critical]` タグを付与してはならない。
   ```yaml
   requirements_checklist:
     - "[critical] <Verify フェーズの必達要件>"
     - "<通常要件>"
     - "<residual_risks[0]>"   # [critical] タグなし
     - "<residual_risks[1]>"   # [critical] タグなし
   ```
2. **`task_context`** には成果物パスリストと背景情報のみを含める。`residual_risks` を `task_context` に含めることは禁止する。
3. **refine-loop の収束判定**は変更しない。`[critical]` タグなし項目の FAIL はループを継続させず `residual_issues` に記録される（ADR-007 / ADR-018 準拠）。
4. **`[critical]` タグの追加禁止**: Caller は Hierarchy の `residual_risks` を `requirements_checklist` に橋渡しする際、タグを昇格させてはならない。Hierarchy の Reviewer が non-critical と判定したリスクを Verify フェーズで blocking に変えることは品質モデルの一貫性を破壊する。

### 変更対象ファイル

| ファイル | 変更内容 |
| :--- | :--- |
| `taskforce/.apm/skills/call-hierarchy/SKILL.md` | Caller Response Contract（全タスク APPROVED 行）の引き継ぎ先を `task_context` → `requirements_checklist` に変更 |
| `easy-agent/.apm/agents/easy-agent.agent.md` | call-refine-loop テンプレートに `residual_risks[N]` 項目例と橋渡し注記を追加 |
| `refine-loop/.apm/skills/call-refine-loop/SKILL.md` | `requirements_checklist` と `task_context` パラメータ説明に ADR-020 ルールを明記 |

### 変更しないもの

- `manager_output.json` の `residual_risks` フィールド定義（変更不要）。
- `refine_loop_output.json` のスキーマ（変更不要）。
- refine-loop の収束判定ロジック（`[critical]` 二段階モデルは変更なし）。
- ADR-018 の Hierarchy 内部ルール（ルール 1〜3）は変更しない。本 ADR はルール 4（Caller の振る舞い）を更新する。

## 結果

### Implement → Verify トレーサビリティの完成

| フェーズ | non-critical FAIL の記録場所 | Verify フェーズでの可視性 |
| :--- | :--- | :--- |
| **Hierarchy Reviewer** | `member_output.risks` | — |
| **Hierarchy Manager** | `manager_output.residual_risks` | — |
| **refine-loop（改善前）** | — | `task_context` に埋もれ、追跡されない |
| **refine-loop（改善後）** | `requirements_checklist`（non-critical） | `residual_issues` に明示的に記録される |

### ADR-018 品質モデルの完全統一

ADR-018 の意図（Implement と Verify で同一の二段階モデルを適用）が、フェーズ境界をまたいで完結する:

```
Hierarchy Reviewer
  → [critical] FAIL → REVISE (ブロック)
  → non-critical FAIL → risks → residual_risks (non-critical、記録のみ)
        ↓ ADR-020 bridge
refine-loop Reviewer
  → [critical] FAIL → ループ継続 (ブロック)
  → non-critical FAIL → residual_issues (non-critical、記録のみ)
```

### コンテキスト予算への影響

`residual_risks` が `task_context`（400トークン上限）から `requirements_checklist`（制約なし）へ移動する。ただし `requirements_checklist` 自体の総量は削減禁止（`[critical]` タグ整合性チェックのため常に完全版を渡す必要がある）。`residual_risks` が多い場合（>5件）は要約して追記することを推奨する。

## 関連 ADR

- [ADR-007](./ADR-007-refine-loop-pattern.md) — refine-loop パッケージ分離（`[critical]` タグ起源）
- [ADR-008](./ADR-008-subagent-return-protocol.md) — サブエージェント返却プロトコル
- [ADR-009](./ADR-009-caller-response-contract-convention.md) — Caller Response Contract Convention
- [ADR-013](./ADR-013-context-budget-protocol.md) — Context Budget Declaration Protocol
- [ADR-018](./ADR-018-critical-severity-hierarchy.md) — [critical] Severity Tagging（本 ADR がルール 4 を更新）
