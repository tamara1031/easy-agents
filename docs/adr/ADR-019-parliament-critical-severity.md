# ADR-019: [critical] Severity Tagging for Parliament Quality Gates

## ステータス

Accepted

## コンテキスト

ADR-007（refine-loop）と ADR-018（Hierarchy）は `[critical]` タグによる二段階収束モデルを確立している。
これにより Hierarchy と refine-loop では「必達条件の FAIL のみが差し戻しを発生させ、努力目標の未達は `residual_risks` として受理する」という品質モデルが実現されている。

一方、Parliament パッケージ（ADR-002）はこの二段階モデルを導入しておらず、チェックリスト全項目を均等に扱っていた。この非対称性は以下の問題を生じさせる：

1. **CRITIQUE の過剰発生**: スタイルガイドへの準拠やドキュメント品質などの努力目標が未達でも `CRITIQUE` スタンスが発生し、`max_rejections` カウントを消費する。本来は設計上の根本的対立を解消するために確保すべき議論機会が浪費される。
2. **品質モデルの不一致**: Hierarchy → refine-loop という実行チェーン全体では `[critical]` を認識するにもかかわらず、その前段の Deliberate フェーズ（Parliament）だけが平坦な収束判定を行う。「Parliament では阻まれなかった non-critical な問題が Hierarchy で突然 REVISE の引き金になる」という不整合が生じうる。
3. **合意コスト の高騰**: `[critical]` タグが存在しない設計では、全員が「大筋は賛成だが細部に soft な懸念を持つ」状態でも収束できない。議論が本質的な解決なしに `MAX_ROUNDS` まで引き延ばされる。

## 決定

**Parliament の全レイヤー（member / chairperson / SKILL.md）に `[critical]` 重要度タグを導入し、ADR-018 と同一の二段階収束モデルに統一する。**

### ルール

1. **チェックリスト作成者（call-parliament を呼び出す easy-agent または人間）** は、必達条件の行頭に `[critical]` を付与する。タグなし項目はベストエフォートとして扱う。
   ```
   - [critical] 提案 API は既存の呼び出し規約と後方互換であること
   - コード例のスタイルガイドに準拠すること
   ```

2. **Member（parliament-member）** はスタンス選択時に `[critical]` の有無を考慮する。
   - `[critical]` 項目が1つでも未達・懸念あり → `CRITIQUE` スタンス（blocking）。`condition_for_approval` に critical 問題のみ列挙する。
   - `[critical]` 項目が全て満足済みで non-critical 項目のみに懸念 → `REVISE` スタンス（non-blocking）。`CRITIQUE` は使わない。
   - `[critical]` タグが1つも存在しないチェックリスト → 従来どおり均等扱い。

3. **Chairperson（parliament-chairperson）** は合意判定を二段階で行う。
   - `[critical]` 項目が全員 satisfied → AGREED を宣言可能（non-critical 未解決は `residual_risks` に記録）。
   - いずれかの `[critical]` 項目が未解決のまま `CRITIQUE` スタンスが残存 → AGREED とみなさない。
   - `max_rejections` カウント対象は `[critical]` 項目への CRITIQUE スタンスに起因するブロックのみとする。

4. **Caller（call-parliament/SKILL.md 呼び出し元）** は `checklist_validation` の `is_critical: true` 項目が全 PASS であることを `APPROVED` の条件とする。`is_critical: false` の FAIL は `residual_risks` として受理し、後続フェーズ（call-hierarchy の `task_context` 等）に引き継ぐ。

### スキーマ変更

| スキーマファイル | 変更内容 |
| :--- | :--- |
| `parliament/.apm/skills/call-parliament/schemas/chairperson_output.json` | `checklist_validation` の各エントリに `is_critical: boolean` フィールドを追加（省略時 false、後方互換） |

### 変更しないもの

- `chairperson_output.status` の enum 値（`AGREED` / `CONVERGED` / `MAX_ROUNDS`）は変更しない（CI ADR-017 lint の対象であり後方互換性を維持）。
- `member_message.json` のスキーマは変更しない（is_critical はメッセージ本文の記述で表現する）。
- `[critical]` タグが1つも存在しないチェックリストの扱い: 従来どおり全項目を均等扱いとする（後退しない）。

## 結果

### 品質モデルの統一

ADR-018 が達成した Hierarchy/refine-loop の統一を Parliament にも拡張し、フレームワーク全体で一貫した二段階収束モデルを実現する。

| レイヤー | 収束条件 | non-critical FAIL の扱い |
| :--- | :--- | :--- |
| **refine-loop** | `[critical]` 項目が2連続で全達成 | `residual_issues` に記録、ループを継続しない |
| **Hierarchy Reviewer** | `[critical]` 項目が全 PASS | `risks` に記録、APPROVE を妨げない |
| **Hierarchy Manager** | Reviewer が APPROVE を返す | `residual_risks` に転記して提出 |
| **Parliament Member** | `[critical]` 項目が全員 satisfied | non-critical は `REVISE` に降格、CRITIQUE にしない |
| **Parliament Chairperson** | `[critical]` 項目が全員 satisfied | `residual_risks` に記録して AGREED を宣言 |

### 差し戻しコスト削減

`max_rejections` の消費がより本質的な設計上の対立（機能要件・セキュリティ・後方互換性の未達）に限定され、スタイル・ドキュメント品質・YAGNI 違反といった non-critical 問題による無駄な CRITIQUE ループが解消される。

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
- [ADR-018](./ADR-018-critical-severity-hierarchy.md) — `[critical]` Severity Tagging for Hierarchy Quality Gates
