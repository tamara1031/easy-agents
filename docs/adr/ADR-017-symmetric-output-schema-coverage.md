# ADR-017: Symmetric Output Schema Coverage for All call-* Skills

## ステータス

Accepted

## コンテキスト

ADR-008 / ADR-009 で確立した Caller Response Contract Convention に基づき、parliament と taskforce は JSON スキーマファイル（`chairperson_output.json`、`manager_output.json`）を持ち、CI が双方向のドリフト検出を行っていた。一方、`advisor` と `refine-loop` は SKILL.md のテキスト記述のみに依拠しており、ステータス語彙の変更が CI で検出されない非対称な状態だった。

具体的なリスク:
- `call-advisor/SKILL.md` の verdict 値（PROCEED / CORRECT / ESCALATE / STOP）を変更しても CI が気づかない
- `call-refine-loop/SKILL.md` のステータス値（CONVERGED / MAX_ITER / ESCALATE / ABORT）を変更しても CI が気づかない
- 新しいステータスを追加する際に SKILL.md とスキーマが乖離するリスク

## 決定

**全 call-* スキルに JSON 出力スキーマファイルを必須化し、CI による双方向 lint を完全適用する。**

### ルール

1. `*/.apm/skills/call-*/schemas/` ディレクトリに少なくとも 1 つの JSON スキーマファイルを配置する
2. スキーマには終端ステータス語彙を `enum` フィールドとして宣言する
3. CI の "Lint subagent output schema status enum consistency" ステップに当該スキーマを登録し、SKILL.md の Caller Response Contract と双方向整合を確認する

### 追加したスキーマ

| スキーマファイル | enum フィールド | 値セット |
| :--- | :--- | :--- |
| `advisor/.apm/skills/call-advisor/schemas/advisor_output.json` | `verdict` | `PROCEED`, `CORRECT`, `ESCALATE`, `STOP` |
| `refine-loop/.apm/skills/call-refine-loop/schemas/refine_loop_output.json` | `status` | `CONVERGED`, `MAX_ITER`, `ESCALATE`, `ABORT` |

### 適用しないケース

- `DISPATCH_FAILURE` はサブエージェント起動失敗時に呼び出し元が生成するメタステータスであり、エージェント自身が返すことはない。スキーマ enum には含めない（ADR-015 参照）。

## 結果

| パッケージ | スキーマ | CI 双方向 lint |
| :--- | :--- | :--- |
| `parliament` | `chairperson_output.json` | ✅（従来から） |
| `taskforce` | `manager_output.json` | ✅（従来から） |
| `advisor` | `advisor_output.json` | ✅（ADR-017 で追加） |
| `refine-loop` | `refine_loop_output.json` | ✅（ADR-017 で追加） |

全 call-* スキルでステータス語彙のシングルソース・オブ・トゥルースが確立され、スキーマと SKILL.md のどちらかに変更が加わった場合に CI が即座に検出できるようになった。

## 関連 ADR

- [ADR-008](./ADR-008-subagent-return-protocol.md) — サブエージェント返却プロトコルの形式化
- [ADR-009](./ADR-009-caller-response-contract-convention.md) — Caller Response Contract Convention
- [ADR-015](./ADR-015-dispatch-failure-protocol.md) — Dispatch Failure Protocol
