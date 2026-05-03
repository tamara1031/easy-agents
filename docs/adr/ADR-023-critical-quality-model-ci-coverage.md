# ADR-023: CI Lint Coverage for [critical] Quality Model and Residual Risks Bridge

## ステータス

Accepted

## コンテキスト

ADR-018（Hierarchy [critical] 重要度タグ）、ADR-019（Parliament [critical] 重要度タグ）、ADR-020（Residual Risks チェックリスト橋渡し）は Pipeline 全体の品質モデルの根幹をなす決定であり、いずれも Accepted ステータスで実装済みである。

しかし、これら 3 つの ADR には対応する CI Lint が存在しなかった。その結果、以下のサイレントな退行が検知されないリスクが生じていた。

1. **スキーマからの `is_critical` 削除**: `member_output.json`・`manager_output.json`・`chairperson_output.json` の `is_critical: boolean` フィールドが削除されても CI は検知しない（ADR-018/019 退行）。
2. **Caller Response Contract の `is_critical` 文書削除**: `call-hierarchy/SKILL.md` または `call-parliament/SKILL.md` の CRC セクションから `is_critical` の説明が消えても CI は検知しない（ADR-018/019 退行）。
3. **residual_risks の橋渡し先変更**: `call-hierarchy/SKILL.md` の記述が `requirements_checklist` から `task_context` に戻っても、また `call-refine-loop/SKILL.md` や `easy-agent.agent.md` の ADR-020 参照が消えても CI は検知しない（ADR-020 退行）。

## 決定

**ADR-018/019/020 が規定した品質モデルに対して、以下の 3 つの CI Lint を追加する。**

### Lint K — `is_critical` スキーマフィールド存在確認（ADR-018/019）

対象スキーマの `checklist_coverage` / `checklist_validation` エントリに `is_critical: boolean` フィールドが存在することを検証する。

| スキーマ | 検証対象 JSON Pointer |
| :--- | :--- |
| `taskforce/.apm/skills/call-hierarchy/schemas/member_output.json` | `.properties.checklist_coverage.additionalProperties.properties.is_critical` |
| `taskforce/.apm/skills/call-hierarchy/schemas/manager_output.json` | `.properties.checklist_validation.items.properties.is_critical` |
| `parliament/.apm/skills/call-parliament/schemas/chairperson_output.json` | `.properties.checklist_validation.additionalProperties.properties.is_critical` |

### Lint L — Caller Response Contract 内 `is_critical` ドキュメント確認（ADR-018/019）

スキーマに `is_critical` が存在しても、呼び出し元が使い方を知らなければ品質モデルは機能しない。`call-hierarchy/SKILL.md` と `call-parliament/SKILL.md` の Caller Response Contract セクションに `is_critical` への言及があることを検証する。

### Lint M — ADR-020 残存リスク橋渡しドキュメント確認（ADR-020）

ADR-020 は Hierarchy の `residual_risks` を refine-loop の `requirements_checklist`（`task_context` ではない）に橋渡しすることを規定する。この橋渡しが 3 つのファイルに正しく文書化されていることを検証する。

| ファイル | 検証内容 |
| :--- | :--- |
| `taskforce/.apm/skills/call-hierarchy/SKILL.md` | `requirements_checklist` が `residual_risks` の橋渡し先として言及されているか |
| `refine-loop/.apm/skills/call-refine-loop/SKILL.md` | `ADR-020` または `residual_risks`+`requirements_checklist` の組み合わせが言及されているか |
| `easy-agent/.apm/agents/easy-agent.agent.md` | `ADR-020` が言及されているか |

## 結果

### 退行保護の完成

| ADR | 内容 | Lint K | Lint L | Lint M |
| :--- | :--- | :--- | :--- | :--- |
| **ADR-018** | Hierarchy `is_critical` タグ | ✅ スキーマ検証 | ✅ CRC 文書検証 | — |
| **ADR-019** | Parliament `is_critical` タグ | ✅ スキーマ検証 | ✅ CRC 文書検証 | — |
| **ADR-020** | residual_risks 橋渡し | — | — | ✅ 橋渡し文書検証 |

### 既存の CI 体制との整合

本 ADR は新しい品質モデルの決定を行うものではなく、既存の ADR-018/019/020 の決定に対する **CI 退行保護の後付け宣言** である。ADR-021（Canonical Source Section Registry）と ADR-022（Agent Invocation Name Coverage）が採用した「ADR 決定 + 対応 Lint」の同時追加パターンに準じて、過去の未保護 ADR を遡及的にカバーする。

### 影響範囲

- 追加ファイル: `.github/workflows/check.yml`（Lint K / L / M の 3 ステップ追加）
- 変更なし: スキーマファイル、SKILL.md ファイル、エージェントファイル（既に ADR-018/019/020 準拠）

## 関連 ADR

- [ADR-018](./ADR-018-critical-severity-hierarchy.md) — [critical] Severity Tagging for Hierarchy Quality Gates
- [ADR-019](./ADR-019-parliament-critical-severity.md) — [critical] Severity Tagging for Parliament Quality Gates
- [ADR-020](./ADR-020-residual-risks-checklist-bridge.md) — Residual Risks Checklist Bridge
- [ADR-021](./ADR-021-canonical-source-section-registry.md) — Canonical Source Section Registry（CI パターンの先例）
- [ADR-022](./ADR-022-agent-invocation-name-coverage.md) — Agent Invocation Name Coverage（CI パターンの先例）
