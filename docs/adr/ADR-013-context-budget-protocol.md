# ADR-013: Context Budget Declaration Protocol (call-* スキルのトークン予算宣言)

- **ステータス**: Accepted
- **決定日**: 2026-04-27

## コンテキスト

easy-agents のマルチエージェントフレームワークでは、easy-agent から始まり Parliament/Hierarchy がサブエージェントを多段に起動する。呼び出し階層は最大4段（easy-agent → Chairperson/Manager → Member → Reviewer）に達し、各段でコンテキストウィンドウの消費が積み重なる。

現状の問題:

1. **コンテキスト予算の非対称な宣言**: `call-advisor/SKILL.md` には「トークン予算」が明示（入力500〜700トークン、出力400〜700トークン）されているが、`call-parliament`・`call-hierarchy`・`call-refine-loop` には同等の予算宣言がない。実装者は各スキルがオーケストレーターのコンテキストを何トークン消費するかを推測に頼るしかない。

2. **コンテキスト爆発の検出不能**: `call-parliament` は「Context Window Management」セクションで圧縮ガイドラインを定めているが、圧縮の前提となる「そもそも何トークンを想定しているか」が明記されていない。オーバーフロー時の段階的縮小ルールも不在。

3. **ADR-009 との非対称性**: ADR-009 は全 `call-*/SKILL.md` に「Caller Response Contract」の宣言を義務付け、CI で強制している。しかしコンテキスト予算は `call-advisor` のみの慣行であり、他スキルへの適用義務がない。この非対称性が将来のスキル追加時に予算宣言の退行を招く。

4. **並列呼び出し時の総量見積もりが困難**: easy-agent が `call-parliament` と `call-hierarchy` を連鎖（Parliament → Plan → Hierarchy）させる際、両スキルのコンテキスト消費を合算した上限見積もりができない。

## 決定

### 1. 全 `call-*/SKILL.md` に Context Window Management セクションを義務付ける

すべての `call-*/SKILL.md` は以下の要素を含む **Context Window Management** (または相当する管理セクション) を持たなければならない。

- **委譲時のコンテキスト最小化ガイドライン**: オーケストレーターがスキルを呼び出す際にどの情報を渡し、何を省略すべきか。
- **呼び出し階層内の爆発防止策**: 中間エージェントが子エージェントへ渡すコンテキストをどう圧縮するか。
- **トークン予算テーブル (`### トークン予算`)**: 各呼び出し段のインプット上限・アウトプット上限を表形式で明示。

### 2. トークン予算テーブルのフォーマット

各 `### トークン予算` サブセクションは以下の列を持つ表を含むこと。

| 列 | 説明 |
| :--- | :--- |
| 階層 | 呼び出し元 → 呼び出し先 のペア |
| 入力上限 | 受け取るプロンプトの想定最大トークン数（内訳コメント付き） |
| 出力上限 | 返却する応答の想定最大トークン数 |

加えて、**超過時の対応**（段階的削減ルール）を注記として記述すること。

### 3. 命名規則

既存の `call-advisor` は `## Context Budget Tracking (コンテキスト管理)` を採用している。
`call-parliament` は `## Context Window Management (コンテキスト管理)` を採用している。
どちらも同義であり、本 ADR は既存の命名を変更しない。

**新規追加・変更時の基準**:
- 新たに追加するセクションは `## Context Window Management (コンテキスト管理)` を優先する。
- 命名の違いは CI lint で両形式を許容する正規表現で吸収する。

### 4. CI による構造検証

`check.yml` に以下の Lint ステップを追加する。

#### Lint E: Context Budget 宣言の存在検証

全 `call-*/SKILL.md` が `Context.*Management`、`Context.*Budget`、`コンテキスト管理`、`コンテキスト予算` のいずれかを含む H2 以上の見出しを持つことを検証する。`トークン予算` サブセクションの存在は推奨だが CI での強制は行わない（内容の正確性は人間レビューに委ねる）。

## 設計の根拠

### なぜ ADR-009 と対称な義務化か

ADR-009 が「Caller Response Contract は呼び出し元が守るべき契約」と定めたように、本 ADR は「Context Budget は呼び出し元が事前に知るべき予算」と定める。両者は「スキルが宣言すべき公約事項」という同じカテゴリに属する。ADR-009 が CI 強制によって退行を防いだように、本 ADR も CI 強制によって新スキル追加時の宣言漏れを防ぐ。

### なぜ具体的なトークン数が必要か

「要約してから渡す」という定性的なガイドラインは既に多くのスキルで記述されているが、「要約した結果が何トークンになるべきか」という定量的な目標がない。具体的な上限があることで、実装者は「500トークン以内に収まっているか」をセルフチェックでき、プロンプトテンプレートの設計時に過剰なコンテキストを事前に削減できる。

### トークン数の根拠

| スキル | 入力上限の根拠 |
| :--- | :--- |
| `call-advisor` | 既存値 (500〜1000) を継承。1対1相談の性格上、簡潔さを優先。 |
| `call-parliament` | 議題1件あたり context 500 + checklist 300 + topic 200 = 1,000トークン上限。 |
| `call-hierarchy` | タスク1件あたり context 500 + checklist 300 + task description 200 = 1,000トークン上限。 |
| `call-refine-loop` | task_context 400 + checklist 400 + subject 200 = 1,000トークン上限。 |

Parliament・Hierarchy は複数の子エージェントを直列起動するため、子エージェント1段あたり 700〜800 トークンのインプット上限とし、子エージェントのアウトプットは要約の後にオーケストレーターへ 500〜600 トークンで報告する。

## 結果として得られる特性

- **予算の可視化**: 全スキルのコンテキスト消費量が事前に把握でき、Parliament + Hierarchy 連鎖時の総量見積もりが可能になる。
- **段階的縮小ルールの統一**: オーバーフロー時の対応が各スキルで標準化され、実装者が個別に考える必要がなくなる。
- **CI による退行防止**: 新スキル追加時に宣言漏れを CI が検出する。
- **ADR-009 との対称性**: Caller Response Contract と並ぶ、スキルの「2大公約事項」として位置付けられる。

## 今後の課題

- `call-advisor` の `Context Budget Tracking` セクション名を `Context Window Management` に統一するか、または両名称を正式に許容する命名ガイドを ADR に追記する。
- トークン予算の実測値収集: 実際の運用データに基づいて各スキルの上限値を調整する。
- easy-agent.agent.md に「並列呼び出し時の総コンテキスト見積もり」セクションを追加する（Parliament + Hierarchy 連鎖時の上限合算ルール）。

## 関連決定

- ADR-009: Caller Response Contract Convention — 同じ「スキルの公約事項」カテゴリとして並立する。
- ADR-001: Advisor パターン — `call-advisor` の既存 Context Budget Tracking が本 ADR の参照実装。
- ADR-002: Parliament モデル — `call-parliament` の Context Window Management セクションが本 ADR 適用対象。
- ADR-003: 階層型タスク実行 — `call-hierarchy` が本 ADR の新規適用対象。
- ADR-007: refine-loop パターン — `call-refine-loop` が本 ADR の新規適用対象。
