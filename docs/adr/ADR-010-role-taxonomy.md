# ADR-010: 役割明確化タクソノミー (Skills / Agents / Instructions / Hooks)

- **ステータス**: Accepted
- **決定日**: 2026-04-27

## コンテキスト

easy-agents-hub のエージェント定義 (`*.agent.md`) は、現状で複数の異質な責務を1ファイルに混在させている。代表例として `easy-agent/.apm/agents/easy-agent.agent.md` (567 行) には:

- エージェント自体のアイデンティティと能力（Overview, Phase Pipeline 等）
- 普遍的な安全ルール（Pre-processing Guards, Confirmation Gates 等）
- 定期的な定型処理（Auto-Memory Protocol — SessionStart リコール、Phase Gate 完了時の memorize 等）

が混在しており、以下の問題が生じている。

1. **責務の不透明化**: あるセクションが「このエージェントの能力」なのか「全エージェントが従うべき普遍ルール」なのか「自動的に発火する定型処理」なのか、形式から判別できない。
2. **再利用性の欠如**: 普遍ルールを別エージェントが参照する手段がない。各エージェントが独自に書き写す or 暗黙にユーザー解釈に委ねるしかない。
3. **保守コストの増大**: 単一ファイルが肥大化し、「このセクションを変更すると何に影響するか」が読み取りづらい。
4. **AI エージェント自身の認知負荷**: easy-agent をロードした Claude Sonnet が 567 行のプロンプトから「自分が今何をすべきか」を毎回切り出すコストが大きい。

## 決定

**コードベース内のすべての設計成果物を以下の 4 役割のいずれかに分類するタクソノミーを公式化する。**

各役割は責務・存在場所・Claude Code プリミティブ対応が明確に異なる。

### 役割 1: Skills

**定義**: エージェントが**持つ能力、実行が許される内容**。

| 観点 | 内容 |
| :--- | :--- |
| 責務 | 特定の能力を「呼び出し可能な単位」として実装する。サブエージェント dispatch・特殊操作・外部ツール起動等。 |
| 存在場所 (本リポジトリ) | `<package>/.apm/skills/<skill-name>/SKILL.md` |
| ビルド成果物 | `.claude/skills/<skill-name>/SKILL.md` |
| Claude Code プリミティブ | Skill (`Skill` ツールで起動) |
| 識別ヒント | frontmatter に `agents: [...]` または明確な操作仕様を持つ。SKILL.md ファイル名で固定。 |

**現コードベースの該当物**: `call-advisor`, `call-parliament`, `call-hierarchy`, `call-refine-loop`, `long-term-memory`

### 役割 2: Agents

**定義**: タスクの**実行をする主体**。実行できる skill がプロンプトで紐づけられる。

| 観点 | 内容 |
| :--- | :--- |
| 責務 | アイデンティティと意思決定ロジックを持ち、ツール・skill を介してタスクを完遂する。 |
| 存在場所 (本リポジトリ) | `<package>/.apm/agents/<agent-name>.agent.md` |
| ビルド成果物 | `.claude/agents/<agent-name>.md` |
| Claude Code プリミティブ | Subagent (`agent` ツールで dispatch) |
| 識別ヒント | frontmatter に `name`, `description`, `model`, `tools` を持つ。`*.agent.md` ファイル名で固定。 |

**現コードベースの該当物**: `easy-agent`, `advisor`, `parliament-chairperson`, `parliament-member`, `hierarchy-manager`, `hierarchy-member`, `refine-loop`

### 役割 3: Instructions

**定義**: **普遍のルール、空気のようなもの**。必要に応じて applyto などで名前空間ごとの空気を定義することも可能。

| 観点 | 内容 |
| :--- | :--- |
| 責務 | 特定エージェントに紐づかない規範・制約・原則。常時有効で、エージェント・スキル・ツール呼び出しの全てに影響する。 |
| 存在場所 (本リポジトリ) | ルート `CLAUDE.md`（普遍）、将来的にはパッケージスコープの `<package>/.apm/instructions/*.md`（applyto 相当） |
| ビルド成果物 | `CLAUDE.md` (Claude Code が SessionStart で読む) |
| Claude Code プリミティブ | Project memory / instructions (frontmatter `applyTo:` で namespace 限定可) |
| 識別ヒント | 「常に〜する」「〜してはいけない」「〜の場合は必ず〜」等の規範文。特定エージェントの能力と独立して有効。 |

**現コードベースの該当物**: ルート `CLAUDE.md`（APM パッケージ管理に関する universal な build/structure ルール）。`easy-agent.agent.md` 内の Pre-processing Guards / Confirmation Gates / 3-Layer Conflict Resolution / Constraints / Overengineering 対策 / Context Window Management 等は本来 Instructions 性が強いが、現状はエージェント定義に同居している。

### 役割 4: Hooks

**定義**: agent が定期的に行う recall や memorize などの**定型作業やルール（記憶や睡眠など）**。基本的には agent hooks を用いるが、誰にでも当てはまるルールは全体に反映される箇所で定義する。

| 観点 | 内容 |
| :--- | :--- |
| 責務 | 特定のイベント（SessionStart / フェーズ遷移 / N exchange 経過 / Tool 実行前後等）で**自動的に発火**する定型処理。条件と動作のペア。 |
| 存在場所 (本リポジトリ) | 現状は agent 定義内に埋め込み。将来的には `<package>/.apm/hooks/<event>.md`（agent hook）またはルート `CLAUDE.md` 内の universal section（誰にでも当てはまる場合）。 |
| ビルド成果物 | `.claude/hooks/<event>` （実行可能 hook）または `CLAUDE.md` 内の hook 仕様（プロンプト的 hook） |
| Claude Code プリミティブ | settings.json hooks (SessionStart, PostToolUse 等) または skill 内 hook spec |
| 識別ヒント | 「〜の時に自動的に〜する」「〜が発生したら〜」等の発火条件 + 動作のペア。エージェントの本来タスクとは独立。 |

**現コードベースの該当物**: `easy-agent.agent.md` の **Auto-Memory Protocol**（SessionStart リコール、Phase Gate 完了時の memorize、15 exchange 経過時のサニティチェック）は完全に hook 形状で、本来 hook として分離すべき内容。

## 「どの役割に属するか」の判定ルール

新規コンテンツ追加時、以下のフローチャートで分類する:

```
コンテンツが「呼び出し可能な能力単位」として独立して使えるか?
  Yes → Skill
  No → ↓
コンテンツが「タスク実行主体としてのアイデンティティ・意思決定」を定義するか?
  Yes → Agent
  No → ↓
コンテンツが「特定イベントで自動発火する条件 + 動作」のペアか?
  Yes → Hook
  No → ↓
コンテンツが「常時有効な規範・原則・制約」か?
  Yes → Instruction
```

### 紛らわしいケースの判定例

| ケース | 分類 | 理由 |
| :--- | :--- | :--- |
| 「Phase Gate で REVISE が 2回続いたら Advisory に相談する」 | Agent (capability) | easy-agent の意思決定ロジック。他エージェントには適用されない。 |
| 「破壊的操作の前には必ずユーザー確認を取る」 | Instruction | 全エージェント・全スキルに適用される普遍ルール。 |
| 「SessionStart で memoir からユーザー文脈をリコールする」 | Hook | 発火条件 (SessionStart) + 動作 (recall) のペア。easy-agent 起動時に自動発火。 |
| 「call-parliament を呼ぶ条件」 | Skill (Caller Response Contract の前提) または Agent (capability) | 呼び出す側の話なので Agent capability。call-parliament 自身ではない。 |
| 「YAGNI 原則: 必要のないレイヤーを追加しない」 | Instruction | 全エージェントの実装フェーズで適用される普遍ルール。 |

## 現コードベースの再分類（参考マッピング）

`easy-agent.agent.md` の 23 個の H2 セクションを本タクソノミーで再分類すると:

| セクション | 役割 | 備考 |
| :--- | :--- | :--- |
| Overview / Task Execution Flow | Agent (identity) | エージェント自体の説明 |
| 3-axis Classification / Phase Pipeline / Phase Gate Protocol / On-demand Advisory / Escalation Criteria / Escalation Handoff Protocol / Subagent Invocation / Advisory 判定後の処理フロー / Parliament + Hierarchy チェーン / Fallback Chain / Delegation Strategy / Verification Criteria | Agent (capability) | easy-agent の意思決定ロジック |
| Pre-processing Guards / Confirmation Gates / 3-Layer Conflict Resolution / 出荷品質の活用 / 過剰エンジニアリング防止 / Context Window Management / Constraints | Instruction (現状は agent-scoped) | 本来は普遍ルール、将来的に CLAUDE.md または `.apm/instructions/` へ分離可能 |
| Auto-Memory Protocol | Hook | 完全に hook 形状（SessionStart recall + Phase Gate save） |

## 設計の根拠

### なぜ4役割か

3 役割（Skills / Agents / Instructions）では Hooks が Instructions に紛れ込み、5 役割以上だと粒度が細かすぎて新規コンテンツ追加時の判断負荷が増える。Skills/Agents/Instructions/Hooks の4区分は、Claude Code プリミティブ（Skill / Subagent / project memory / settings hooks）と1対1対応しており、ハーネス側のメンタルモデルとも整合する。

### なぜ「即座の物理的分離」を本 ADR ではしないのか

Auto-Memory Protocol を `.apm/hooks/` に物理的に分離する案も検討したが:

1. APM が `.apm/hooks/` をどう扱うかの仕様確認が必要（リスク）
2. 1ファイルから抽出すると現エージェントが「何を読めば自分の hook を理解できるか」のパスが変わる（保守性）
3. タクソノミーが定着する前に物理レイアウトを変えると、別の境界線で再分割したくなる可能性

本 ADR は **タクソノミーの確立とアノテーション** に絞り、物理的な分離・抽出は将来の独立 ADR として記録する。今すぐの価値は「混在状態を可視化する」ことにある。

## 結果として得られる特性

- **役割の自己記述性**: コンテンツが自分の役割を frontmatter / 見出しタグで宣言するため、新規コントリビューター・AI エージェント双方の認知負荷が低下
- **責務分離の指針**: 「どこに書くべきか」迷ったときに4分岐フローチャートで判断可能
- **将来の物理分離への布石**: タクソノミーが先にあれば、後の hooks/ ディレクトリ追加・instructions/ 整備時に境界線が動かない

## 今後の課題

- `easy-agent.agent.md` の Auto-Memory Protocol を `.apm/hooks/` または同等の hook 仕様ファイルへ物理的に抽出する（独立 ADR で扱う）
- ルート `CLAUDE.md` を「universal Instructions」として再定義し、`Pre-processing Guards` 等の現 agent-scoped instruction を移管する
- `.apm/instructions/` のディレクトリ規約と APM 取扱仕様の確定（`applyTo:` 相当の namespace スコープ）
- Hook の発火イベント語彙の標準化（SessionStart / PhaseGateComplete / NExchangeElapsed 等）

## 関連決定

- ADR-005: APM パッケージング — `.apm/` がソースで `.claude/` がビルド成果物という構造の上に本タクソノミーが乗る
- ADR-009: Caller Response Contract Convention — Skill 役割の双方向コントラクト規約。本 ADR の Skills 区分の運用を補強する
