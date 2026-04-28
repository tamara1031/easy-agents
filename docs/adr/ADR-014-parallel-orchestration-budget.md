# ADR-014: Parallel Orchestration Budget (連鎖オーケストレーション予算)

- **ステータス**: Accepted
- **決定日**: 2026-04-28

## コンテキスト

ADR-013 は全 `call-*/SKILL.md` に対してスキル内の各階層のトークン予算テーブルを義務付けた。これにより、Parliament の Chairperson → Member 間・Hierarchy の Manager → Planner/Implementer/Reviewer 間など、**スキル内**のコンテキスト消費量は定量化された。

しかし ADR-013 の「今後の課題」に明記されたように、**スキル間** の予算が未定義のままである。

### 未解決の問題

easy-agent が `designExecute` タスクを処理する際、次の連鎖が発生する。

```
easy-agent
  └─→ call-parliament  (Deliberate フェーズ)
        └─ chairperson_output.json が返却される
  └─→ call-hierarchy   (Implement フェーズ)
        └─ 議会の合意案を context として受け取る
```

このチェーン全体で以下の問題が生じる。

1. **引き継ぎ予算の欠如**: Parliament が返した `chairperson_output.json`（議題1件 = 約600トークン）を、そのまま Hierarchy の `context` 引数（上限500トークン）に渡すと **超過が確実**。しかし easy-agent には「どう圧縮するか」のルールがない。
2. **連鎖総量の見積もり不能**: easy-agent のコストガード（Guard 2）は `estimated_cost = High` とするよう ADR 定義されているが、実際の総トークン消費量の上限を計算する方法がない。
3. **easy-agent の Context Window Management の空洞化**: execution-policy.md の同セクションは定性ルール3行のみであり、ADR-013 が各スキルに義務付けたトークン予算テーブルに相当するものが存在しない。

## 決定

### 1. Cross-Skill Handoff Compression の義務化

easy-agent は Parliament → Hierarchy 連鎖時に **Handoff Compression** ステップを実行しなければならない。

Handoff Compression の手順:
1. `chairperson_output.json` の各議題から **決定事項 (decision)** と **残存リスク (residual_risks)** のみを抽出する。
2. 抽出した内容を箇条書き形式に変換し、**500トークン以内**に収まるよう削減する。
3. 削減後の要約を Hierarchy への `context` 引数として渡す（議長の内部議論・メンバー発言は含めない）。

### 2. 連鎖トークン予算テーブルの策定

easy-agent の Context Window Management セクションに、以下の **連鎖予算テーブル** を追加する。

| 連鎖フェーズ | 予算区分 | 入力上限 | 出力上限 |
| :--- | :--- | :--- | :--- |
| easy-agent → call-parliament | Deliberate 委譲 | 1,000トークン | 600トークン × N議題 |
| Parliament → easy-agent (引き継ぎ前圧縮) | Handoff Compression | 600トークン × N議題 | **500トークン** (決定事項のみ) |
| easy-agent → call-hierarchy | Implement 委譲 | 1,000トークン (context: 500, checklist: 300, task: 200) | 500トークン |
| **連鎖合計 (N=2議題)** | designExecute 最悪ケース | **3,200トークン** | **1,600トークン** |

> **連鎖合計の計算根拠**: Parliament 委譲 1,000 + Parliament 出力圧縮前 1,200 + Hierarchy 委譲 1,000 = 3,200トークン入力。Parliament 出力 1,200 (圧縮後500) + Hierarchy 出力 500 + 中間要約 100 = 1,600トークン出力 (概算)。N議題が3以上になる場合は線形に増加することに注意。

### 3. N議題への対応

N議題がある場合の圧縮上限:

| N議題 | Parliament 生出力 | Handoff 圧縮後 | Hierarchy context 上限 |
| :--- | :--- | :--- | :--- |
| 1 | ≤ 600トークン | ≤ 300トークン | 500トークン内に収容可能 |
| 2 | ≤ 1,200トークン | ≤ 500トークン | **上限ギリギリ**。議題ごとに100〜200トークンで圧縮 |
| 3 | ≤ 1,800トークン | ≤ 500トークン | **圧縮必須**。決定事項のみ60〜100トークン/議題 |
| 4以上 | > 2,400トークン | ≤ 500トークン | **Advisory 相談推奨**。分割実行または summary_only モードを検討 |

### 4. easy-agent の Context Window Management セクション更新の義務化

`easy-agent/.apm/instructions/execution-policy.md` の `## Context Window Management` セクションに、本 ADR が定める **連鎖予算テーブル** と **Handoff Compression ルール** を追記しなければならない。

### 5. CI による構造検証 (Lint F)

`check.yml` に以下の Lint ステップを追加する。

#### Lint F: 連鎖予算テーブルの存在検証

`easy-agent/.apm/agents/easy-agent.agent.md` の Context Window Management セクション内に、`call-parliament` と `call-hierarchy` の両方を含む **連鎖予算テーブル** （H3以上の見出しで `連鎖` または `chain` または `Handoff` を含む）が存在することを検証する。

## 設計の根拠

### なぜ easy-agent 側に Handoff Compression を置くか

各スキルは自分のスキル内部の予算を管理する責任を持つ（ADR-013）。しかし**スキル間の引き継ぎ**はオーケストレーター (easy-agent) の責任領域である。ADR-009 の Relay Principle が「フォールバック選択肢はそのまま転送する」と定めたように、「コンテキストの変換・圧縮」もオーケストレーターが担う。

### なぜ500トークンが Handoff Compression の目標か

Hierarchy の `context` 引数上限は500トークン（ADR-013 / call-hierarchy SKILL.md に明示）。Handoff Compression の目標はこの制約を満たすこと。議題が2件なら1件あたり250トークン、3件なら166トークンで「決定事項 + 残存リスク」を記述できる分量として設定した。

### なぜ議題4件以上で Advisory 相談推奨か

4議題以上（= Parliament 出力 > 2,400トークン → 圧縮後 125トークン/議題以下）では、決定事項のエッセンスのみを Hierarchy に伝えることになり、実装品質が低下するリスクが高い。この場合は Hierarchy への委譲を複数バッチに分割するか、Advisory に判断を仰ぐべき。

## 結果として得られる特性

- **連鎖予算の可視化**: designExecute チェーン全体のトークン消費量を事前に見積もれる。
- **Handoff Compression の標準化**: Parliament → Hierarchy 引き継ぎの圧縮ルールが一箇所に集約される。
- **CI による退行防止 (Lint F)**: easy-agent.agent.md に連鎖予算テーブルが欠落したときに CI が検出する。
- **ADR-013 の完成**: ADR-013 が「今後の課題」とした「easy-agent の並列呼び出し予算」の欠落を解消する。

## 今後の課題

- Hierarchy の複数タスク並列実行時（parallelism > 1）の総コンテキスト消費量の算出ルール。
- Parliament と Hierarchy を並列（非連鎖）で起動するケースの予算定義（現状 designExecute フローでは発生しないが将来のフロー拡張時に検討）。
- N議題 ≥ 4 の分割実行戦略を call-parliament SKILL.md に追記するかどうか。

## 関連決定

- ADR-013: Context Budget Declaration Protocol — 各スキル内の予算定義。本 ADR はその「スキル間予算」補完。
- ADR-009: Caller Response Contract Convention — Relay Principle の定義。
- ADR-003: 階層型タスク実行 — Hierarchy の PIR サイクル。
- ADR-002: Parliament モデル — Chairperson output フォーマット。
