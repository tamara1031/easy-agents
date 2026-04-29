# ADR-018: [critical] Severity Tagging for Hierarchy Quality Gates

## ステータス

Accepted

## コンテキスト

refine-loop パッケージ（ADR-007）は要件チェックリストに `[critical]` タグを導入し、**「`[critical]` 項目が2連続で全達成 → CONVERGED」** という二段階収束モデルを確立している。タグなし（non-critical）項目の FAIL はループを継続させず、残存 issues として記録されるにとどまる。

一方、Hierarchy パッケージ（ADR-003）は全チェックリスト項目を均等に扱っており、スタイルガイドの軽微な不一致や将来的な改善余地のコメント不足といった non-critical な問題でも `REVISE` が発生し、内部フィードバックループが不必要に延長されていた。

この非対称性は以下の問題を引き起こす:

1. **REVISE の過剰発生**: blocking でない軽微な問題が `max_rejections` カウントを消費し、本質的な失敗への許容枠を狭める。
2. **品質モデルの不一致**: refine-loop（Verify フェーズ）は critical/non-critical を区別するが、Hierarchy（Implement フェーズ）は区別しない。上流で許容されたはずの問題が下流で blocking となる逆転が生じうる。
3. **シグナル・ノイズ比の低下**: `residual_risks` が空のまま APPROVE されるか、軽微問題が差し戻し理由に混在するか、の二択しかない。

## 決定

**Hierarchy の全レイヤー（member / manager / SKILL.md）に `[critical]` 重要度タグを導入し、refine-loop と同一の二段階収束モデルに統一する。**

### ルール

1. **チェックリスト作成者（call-hierarchy を呼び出す easy-agent または人間）** は、必達条件の行頭に `[critical]` を付与する。タグなし項目はベストエフォートとして扱う。
   ```
   - [critical] ユーザー認証が機能すること
   - ドキュメントのスタイルガイドに準拠すること
   ```
2. **Reviewer（hierarchy-member）** は各項目の `[critical]` タグ有無を `is_critical` フィールドに記録する。
   - `[critical]` 項目が **全て PASS** → `verdict: "APPROVE"`。non-critical FAIL は `risks` に記録する。
   - `[critical]` 項目が **1 つでも FAIL** → `verdict: "REVISE"`。`rejection_instructions` には critical FAIL のみを列挙する。
3. **Manager（hierarchy-manager）** は Reviewer の APPROVE 時に Reviewer の `risks`（non-critical FAIL）を `residual_risks` へ転記し、オーケストレーターへ提出する。差し戻し枯渇カウントの対象は `[critical]` 項目の FAIL に起因する REVISE のみ。
4. **Caller（call-hierarchy/SKILL.md 呼び出し元）** は `checklist_validation` の `is_critical: true` 項目が全 PASS であることを `APPROVED` の条件とする。`is_critical: false` の FAIL は `residual_risks` として受理し、Verify フェーズ（refine-loop）の `task_context` に引き継ぐ。

### スキーマ変更

| スキーマファイル | 変更内容 |
| :--- | :--- |
| `taskforce/.apm/skills/call-hierarchy/schemas/member_output.json` | `checklist_coverage` の各エントリに `is_critical: boolean` フィールドを追加 |
| `taskforce/.apm/skills/call-hierarchy/schemas/manager_output.json` | `checklist_validation` の各エントリに `is_critical: boolean` フィールドを追加 |

### 変更しないもの

- `manager_output.status` の enum 値（`IN_REVIEW` / `ERROR`）は変更しない（CI ADR-017 lint の対象であり後方互換性を維持）。
- `member_output.verdict` の enum 値（`DONE` / `APPROVE` / `REVISE`）は変更しない。
- `[critical]` タグが1つも存在しないチェックリストの扱い: 従来どおり全項目を均等扱いとする（後退しない）。

## 結果

### 品質モデルの統一

| レイヤー | 収束条件 | non-critical FAIL の扱い |
| :--- | :--- | :--- |
| **refine-loop** | `[critical]` 項目が2連続で全達成 | `residual_issues` に記録、ループを継続しない |
| **Hierarchy Reviewer** | `[critical]` 項目が全 PASS | `risks` に記録、APPROVE を妨げない |
| **Hierarchy Manager** | Reviewer が APPROVE を返す | `residual_risks` に転記して提出 |

### 差し戻しコスト削減

`max_rejections` の消費がより本質的な失敗（機能要件の未達など）に限定され、スタイル・ドキュメント・YAGNI 違反といった non-critical な問題による無駄な REVISE ループが解消される。

### 後方互換性

`is_critical` フィールドはオプション（`"type": "boolean"` 、必須ではない）のため、既存の Manager/Reviewer 実装に対するスキーマ互換性は維持される。`[critical]` タグなしのチェックリストは従来どおり動作する。

## 関連 ADR

- [ADR-003](./ADR-003-hierarchy-execution.md) — 階層型タスク実行 (PIR サイクル)
- [ADR-007](./ADR-007-refine-loop-pattern.md) — refine-loop の `[critical]` タグ起源
- [ADR-008](./ADR-008-subagent-return-protocol.md) — サブエージェント返却プロトコル
- [ADR-009](./ADR-009-caller-response-contract-convention.md) — Caller Response Contract Convention
- [ADR-017](./ADR-017-symmetric-output-schema-coverage.md) — JSON スキーマ適用（status enum は変更なし）
