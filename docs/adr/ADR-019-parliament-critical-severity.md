# ADR-019: [critical] Severity Tagging for Parliament Quality Gates

## ステータス

Accepted

## コンテキスト

ADR-018 は Hierarchy パッケージ（Implement フェーズ）に `[critical]` 重要度タグを導入し、refine-loop（Verify フェーズ）と同一の二段階収束モデルに統一した。しかし Parliament パッケージ（Deliberate フェーズ）は依然として全チェックリスト項目を均等に扱っており、以下の非対称性が残存していた。

| フェーズ | パッケージ | [critical] サポート |
| :--- | :--- | :--- |
| Deliberate | parliament | ❌ なし |
| Implement | taskforce | ✅ ADR-018 で追加 |
| Verify | refine-loop | ✅ ADR-007 から存在 |

この非対称性は以下の問題を引き起こす:

1. **CRITIQUE の過剰発生**: スタイルガイドへの準拠やドキュメント品質などの努力目標が未達でも `CRITIQUE` スタンスが発生し、`max_rejections` カウントを消費する。本来は設計上の根本的対立を解消するために確保すべき議論機会が浪費される。
2. **フェーズ間品質モデルの不一致**: 上流の Parliament が non-critical FAIL で REJECTED を出す一方、下流の Hierarchy は同じ non-critical FAIL を `residual_risks` として受理してしまう。上流が厳しく下流が緩い逆転が生じうる。
3. **`max_rejections` 枠の浪費**: ブロッキングでない軽微問題が差し戻し枠を消費し、本質的な合意の失敗への許容枠を狭める。

## 決定

**Parliament の全レイヤー（member / chairperson / SKILL.md）に `[critical]` 重要度タグを導入し、refine-loop・Hierarchy と同一の二段階収束モデルに統一する。**

### ルール

1. **チェックリスト作成者（call-parliament を呼び出す easy-agent または人間）** は、合意必達の条件の行頭に `[critical]` を付与する。タグなし項目はベストエフォートとして扱う。
   ```
   - [critical] 競合アプローチから1つの設計方針に絞り込まれていること
   - APIドキュメントのサンプルが含まれていること
   ```

2. **Member（parliament-member）** はスタンス選択時に `[critical]` の有無を考慮する。
   - `[critical]` 項目が1つでも未達・懸念あり → `CRITIQUE` スタンス（blocking）。`condition_for_approval` に critical 問題のみ列挙する。
   - `[critical]` 項目が全て満足済みで non-critical 項目のみに懸念 → `REVISE` スタンス（non-blocking）。`CRITIQUE` は使わない。
   - `[critical]` タグが1つも存在しないチェックリスト → 従来どおり均等扱い。

3. **Chairperson（parliament-chairperson）** は合意判定を二段階で行う。
   - `[critical]` 項目が全て PASS → `AGREED` を宣言可能（non-critical 未解決は `residual_risks` に記録）。
   - `[critical]` 項目が1つでも FAIL → 合意未達（ラウンド継続、または MAX_ROUNDS でも REJECTED 扱い）。
   - `max_rejections` カウント対象は `[critical]` 項目への CRITIQUE スタンスに起因するブロックのみとする。

4. **Caller（call-parliament/SKILL.md 呼び出し元）** は `checklist_validation` の `is_critical: true` 項目が全 PASS であることを `APPROVED` の条件とする。`is_critical: false` の FAIL は `residual_risks` として受理し、後続フェーズ（call-hierarchy の `task_context` 等）に引き継ぐ。

### スキーマ変更

| スキーマファイル | 変更内容 |
| :--- | :--- |
| `parliament/.apm/skills/call-parliament/schemas/chairperson_output.json` | `checklist_validation` の各エントリ（additionalProperties）に `is_critical: boolean` フィールドを追加（optional、省略時 false、後方互換） |

### 変更しないもの

- `chairperson_output.status` の enum 値（`AGREED` / `CONVERGED` / `MAX_ROUNDS`）は変更しない（CI ADR-017 lint の対象であり後方互換性を維持）。
- `member_message.json` のスキーマは変更しない（is_critical はメッセージ本文の記述で表現する）。
- `[critical]` タグが1つも存在しないチェックリストの扱い: 従来どおり全項目を均等扱いとする（後退しない）。

## 結果

### 品質モデルの完全統一

| フェーズ | パッケージ | 収束条件 | non-critical FAIL の扱い |
| :--- | :--- | :--- | :--- |
| **Deliberate** | parliament | `[critical]` 項目が全 PASS | `residual_risks` に記録、REJECTED を引き起こさない |
| **Implement** | taskforce | `[critical]` 項目が全 PASS | `risks` に記録、APPROVE を妨げない（ADR-018） |
| **Verify** | refine-loop | `[critical]` 項目が 2 連続で全達成 | `residual_issues` に記録、ループを継続しない（ADR-007） |

Pipeline 全体（Deliberate → Implement → Verify）で `[critical]` の意味論が統一され、フェーズをまたぐ品質評価の一貫性が確保される。

### 差し戻しコスト削減

`max_rejections` の消費がより本質的な設計上の対立（機能要件・セキュリティ・後方互換性の未達）に限定され、スタイル・ドキュメント品質・YAGNI 違反といった non-critical 問題による無駄な CRITIQUE/REJECTED ループが解消される。

### Parliament → Hierarchy 引き継ぎの改善

Parliament の non-critical FAIL が `residual_risks` として明示されることで、Hierarchy の `context` に構造的に引き継ぐことができる。Hierarchy 側はこれを `residual_risks` として受け取り、refine-loop の `task_context` へさらに伝播させる。

### 後方互換性

- `is_critical` フィールドはオプション（省略時は false）のため、既存の Chairperson 実装に対するスキーマ互換性は維持される。
- `[critical]` タグなしのチェックリストは従来どおり動作する。
- `chairperson_output.status` の enum 値は変更しないため CI lint は引き続き通過する。

## 関連 ADR

- [ADR-002](./ADR-002-parliament-model.md) — Parliament モデル（多視点合意形成）
- [ADR-007](./ADR-007-refine-loop-pattern.md) — refine-loop の `[critical]` タグ起源
- [ADR-008](./ADR-008-subagent-return-protocol.md) — サブエージェント返却プロトコル
- [ADR-009](./ADR-009-caller-response-contract-convention.md) — Caller Response Contract Convention
- [ADR-017](./ADR-017-symmetric-output-schema-coverage.md) — JSON スキーマ適用（status enum は変更なし）
- [ADR-018](./ADR-018-critical-severity-hierarchy.md) — `[critical]` Severity Tagging for Hierarchy Quality Gates（本 ADR の先行決定）
