# ADR-011: Hook Specification Format & Trigger Event Vocabulary

- **ステータス**: Accepted
- **決定日**: 2026-04-27
- **関連**: [ADR-010](./ADR-010-role-taxonomy.md) (役割タクソノミー)

## コンテキスト

[ADR-010](./ADR-010-role-taxonomy.md) は設計成果物を **Skills / Agents / Instructions / Hooks** の4役割に分類するタクソノミーを確立した。しかし `[role: hook]` とタグ付けされたコンテンツがどのような形式で記述されるべきかは未定義であり、結果として:

1. **発火条件の表現が不統一**: `easy-agent.agent.md` の Auto-Memory Protocol では「セッション開始時」「Phase Gate の verdict が APPROVED で…」「15 exchange 以上が経過」など、**散文・表・条件式が混在** している。
2. **`{trigger, action}` の対応が読み解きにくい**: 1つの hook（例: `feedback` 保存）の発火条件が複数のセクションに散らばっており、機械処理はもちろん、人間レビュアーも「いつ何が起きるか」を一発で理解できない。
3. **物理抽出のブロッカー**: ADR-010 の「今後の課題」で `.apm/hooks/` への物理抽出が挙げられているが、**抽出先のファイル形式が決まっていない** ため、抽出を始めると形式議論で停滞する。

本 ADR はこの3点に対し、**Hook の正規仕様形式と発火イベント語彙の closed set** を定める。物理ディレクトリレイアウト（`.apm/hooks/<name>.hook.md` など）は本 ADR ではスコープ外とし、別 ADR で扱う。

## 決定

**Hook は `{event, condition, action, scope}` の 4 タプルとして記述する。** 各フィールドは以下に定める語彙・形式に従う。

### Hook Specification 4-Tuple

| フィールド | 型 | 必須 | 説明 |
| :--- | :--- | :--- | :--- |
| `event` | enum (下記語彙) | ✅ | 発火イベント。closed set から選択。 |
| `condition` | 述語式 (自然言語可) | ✅ | event 発火時に hook が実際に動作する条件。常時実行なら `always`。 |
| `action` | 動作記述 (動詞句) | ✅ | hook が実行する処理。複数ステップなら箇条書き。 |
| `scope` | enum: `agent` \| `universal` | ✅ | hook の適用範囲。`agent` は特定 agent 内のみ、`universal` は全 agent / 全 skill に適用。 |

### Trigger Event Vocabulary (closed set)

新規 hook を追加する際は以下の語彙から `event` を選ぶ。語彙にない発火タイミングが必要な場合は、本 ADR を更新して追加する（CI で語彙外の event は lint エラーとする）。

| Event ID | 発火タイミング | 引数 / 修飾子 |
| :--- | :--- | :--- |
| `SessionStart` | エージェントセッション開始時に1度だけ | なし |
| `SessionEnd` | エージェントセッション終了時に1度だけ | なし |
| `PhaseGateComplete` | Phase Gate 評価が完了した瞬間 | `verdict ∈ {APPROVED, REVISE, DELEGATE, LOOPBACK, ESCALATE, STOP}` |
| `PhaseTransition` | フェーズ N から N+1 への遷移直前 | `from`, `to` |
| `NExchangeElapsed` | 前回チェックから N 回 exchange が経過 | `N: int` |
| `OnExchange` | exchange ごと（毎回） | なし |
| `PreToolUse` | 任意ツール呼び出し直前 | `tool: string` (省略時は全ツール) |
| `PostToolUse` | 任意ツール呼び出し直後 | `tool: string` (省略時は全ツール) |
| `OnUserMessage` | ユーザーメッセージ受信時 | なし |

> **語彙の境界**: Phase Gate 系イベント (`PhaseGateComplete`, `PhaseTransition`) は **easy-agent の Phase Pipeline コンテキストでのみ意味を持つ**。Phase Pipeline を持たないエージェント（advisor 等）は `SessionStart` / `OnExchange` 等の generic イベントのみ使用する。

### Hook Spec の記述形式

Hook はソースファイル内で以下の YAML-like ブロック、または等価な表として記述する。表形式と YAML-like ブロックは意味的に等価で、文書性に応じて使い分ける。

#### Form A (YAML-like, 単一 hook 向け):

```yaml
hook:
  event: PhaseGateComplete
  condition: "verdict == APPROVED AND project state changed"
  action: |
    Skill ツールで long-term-memory の Save を呼び出し、
    items=[{text, tags:['project']}], dedup=true で保存する。
  scope: agent
```

#### Form B (表, 複数 hook の俯瞰向け):

| event | condition | action | scope |
| :--- | :--- | :--- | :--- |
| `SessionStart` | `always` | memoir の Search で過去の `user`/`user-pref`/`feedback`/`project` を recall | agent |
| `PhaseGateComplete[verdict=APPROVED]` | project state が前回保存と差分あり | Save items=[...] tags=['project'] | agent |
| `NExchangeElapsed[N=15]` | 直前の exchange に保存対象がある | 有機学習を確認し、価値あるものだけ Save | agent |

> いずれの形式でも、`event`・`condition`・`action`・`scope` の 4 フィールドを欠かさないこと。

### 推論ルール: scope の決定

Hook の scope は以下の順で決定する:

1. その hook が**特定 agent のアイデンティティ・能力に依存する**ならば `agent`。
2. その hook が**任意 agent / 任意 skill の実行で常に有効**であるべきなら `universal`。

Auto-Memory Protocol は「easy-agent の memoir 統合」という agent 固有能力に依存するため `scope: agent`。一方、「破壊的操作の前にユーザー確認を取る」のような Confirmation Gate は `[role: instruction]` であって hook ではない（発火イベントが「特定 tool 実行前」とすれば hook 化も可能だが、現状 ADR-010 では Instruction として整理されている）。

## 「どこに書くか」の現状ルール

物理ディレクトリ規約は本 ADR ではスコープ外とするが、**当面の運用ルール** を以下に定める:

- **agent-scoped hook**: 該当 agent の `*.agent.md` 内に H2 セクションとして記述し、見出しに `[role: hook]` タグを付ける。Hook spec は本 ADR の Form A または B で記述する。
- **universal hook**: ルート `CLAUDE.md` 内の専用セクション（将来追加）または `.apm/instructions/` 配下（ADR-010 の今後の課題で扱う）。**現時点では universal hook は本リポジトリに存在しない** ため、追加時に本 ADR を更新する。

## CI Lint との関係

本 ADR で定めた語彙・形式は、以下の CI lint で機械的に検証される（実装は段階的に拡充）:

- **role tag 語彙の closed set 検証**: `[role: agent identity | agent capability | instruction | hook]` 以外の値を検出（ADR-010 の closed set を強制）。
- **event 語彙の closed set 検証** (将来): hook spec ブロック内の `event:` フィールドが本 ADR の語彙に含まれることを検証。
- **hook spec の必須フィールド検証** (将来): `[role: hook]` タグの付いたセクション内に `event` / `condition` / `action` / `scope` の4フィールドが揃っていることを検証。

## 設計の根拠

### なぜ4タプルか

3 タプル（event/action/scope）では condition を action に埋め込む必要があり、「何が起きるか」と「いつ起きるか」が混ざる。5 タプル以上（priority・retry policy・timeout 等を分離）は本リポジトリの hook 規模では過剰。Phase Gate Protocol 等で実際に必要になった時点で追加する。

### なぜ event 語彙を closed set にするか

Open vocabulary では「SessionStart」「session-start」「on session start」のような表記揺れが発生し、機械検証が困難になる。closed set + ADR 更新による拡張という運用は、Phase Gate Protocol の verdict 集合 (`APPROVED, REVISE, DELEGATE, LOOPBACK, ESCALATE, STOP`) や Caller Response Contract のステータス集合と同じパターンで一貫性がある。

### なぜ物理ディレクトリ規約を本 ADR に含めないか

ADR-010 の「即座の物理的分離をしない理由」と同じ。形式が固まる前に物理レイアウトを決めると、抽出後に「別の境界線で再分割したい」になりやすい。本 ADR は **形式と語彙** を確定し、抽出は別 ADR で扱う。

## 結果として得られる特性

- **hook の自己記述性**: 4タプルが揃っているので「いつ何が起きるか」が一目で分かる。
- **物理抽出の準備完了**: 抽出時はセクションをファイルに切り出すだけで形式変換不要。
- **機械検証可能性**: 語彙が closed set なので CI で誤記・タイプミスを検出できる。
- **将来拡張への布石**: universal hook、PreToolUse/PostToolUse 駆動の自動化等に拡張する際、形式が決まっているので議論なしで追加できる。

## 今後の課題

- `.apm/hooks/<name>.hook.md` 物理ディレクトリ規約の確定（ADR-010 今後の課題と統合し、別 ADR で扱う）。
- universal hook の表現場所（ルート `CLAUDE.md` vs `.apm/instructions/`）の決定。
- Phase Gate verdict / Caller Response Contract ステータスとの語彙統合（重複定義の解消）。
- hook spec の `condition` 述語式の文法標準化（現状は自然言語可）。

## 関連決定

- **ADR-010**: 役割明確化タクソノミー — 本 ADR は ADR-010 の Hook 役割の運用形式を補強する。
- **ADR-009**: Caller Response Contract Convention — 本 ADR と同じく「closed set + 機械検証」のパターンを踏襲。
- **ADR-006**: Phase Gate Protocol — `PhaseGateComplete` event の発火タイミングを定義する上位プロトコル。
