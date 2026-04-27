# ADR-009: Caller Response Contract Convention の確立

- **ステータス**: Accepted
- **決定日**: 2026-04-27

## コンテキスト

ADR-008 で `call-refine-loop/SKILL.md` に「呼び出し元の応答コントラクト (Caller Response Contract)」を導入し、その後の作業で `call-parliament` / `call-hierarchy` / `call-advisor` の3スキルにも同パターンを展開した。これにより全サブエージェントスキル（4種）が「サブエージェント出力 ↔ 呼び出し元応答」の双方向コントラクトを持つ状態となった。

しかし、この展開過程で以下の繰り返しパターンが暗黙化されたまま各 SKILL.md に分散している。

1. **Two-layer return の使い分け**: `parliament` / `hierarchy` は「per-topic/per-task」と「orchestrator-aggregate」の2層、`refine-loop` / `advisor` は単一階層。これは恣意的選択ではなく、サブエージェント自体の構造（議題分割の有無、純分析か実装か）から導かれる。
2. **Relay Principle の暗黙化**: `max_rejections` 超過などのフォールバック時、呼び出し元（easy-agent）は **サブエージェントが提示した選択肢をそのままユーザーへ転送** する。独自の選択肢を作らない。これは ADR-008 で示唆されたが「原則」として明文化されていなかった。
3. **Single Source of Truth (SSoT) の曖昧さ**: `easy-agent.agent.md` の Fallback Chain と各 SKILL.md の Caller Response Contract が同じ内容を記述している。ドリフトした場合にどちらが正かのルールが欠落。
4. **MAX_ITER vs ESCALATE / CORRECT vs ESCALATE の境界条件**: 「時間切れ vs 設計問題」「修正可能 vs 委譲必要」の判定軸が各 SKILL.md でアドホックに記述されている。

これらの規約を ADR で公式化することで、将来追加されるサブエージェントスキル（例: `call-empirical-prompt-tuning` を APM パッケージ化する場合など）が同じ規約に従って設計されることを保証できる。

## 決定

**Caller Response Contract Convention を SKILL.md の必須セクションとして公式化し、4つの規約を定める。**

### 規約 1: Caller Response Contract セクションは必須

全ての `call-X/SKILL.md` は「呼び出し元の応答コントラクト (Caller Response Contract)」セクションを持たなければならない。最小要件:

- サブエージェントの返却ステータス（または verdict）を網羅する
- 各ステータスごとに「意味」と「呼び出し元が取るべきアクション」を明示する
- 混同されやすいステータス対の境界条件を `>` ブロックで補足する

### 規約 2: Two-layer return は構造から導く

サブエージェントが複数の処理単位（議題、タスク等）を持つ場合は2層構造で記述する。それ以外は単一階層で良い。

| サブエージェントの構造 | 階層 | 例 |
| :--- | :--- | :--- |
| 単一処理（純分析・反復改善） | 単一階層 | `advisor`, `refine-loop` |
| 複数処理単位 + 集約 | per-unit + orchestrator-aggregate | `parliament`, `hierarchy` |

### 規約 3: Relay Principle

呼び出し元は、サブエージェントが提示したフォールバック選択肢を**そのままユーザーへ転送する**。独自の選択肢を作ってはならない。

- ✅ 良い: `max_rejections` 超過時に call-parliament が提示した3択をそのままユーザーに渡す
- ❌ 悪い: easy-agent が独自に「全部やり直す」「無視して進める」等の選択肢を追加する

これによりサブエージェントのフォールバック戦略とオーケストレーターの応答が矛盾しない。

### 規約 4: Single Source of Truth

`easy-agent.agent.md` の Fallback Chain と各 `call-X/SKILL.md` の Caller Response Contract がドリフトした場合、**SKILL.md が正**とする。

- 理由 1: SKILL.md はサブエージェントの実装と同じパッケージにあり、変更時に同時に更新されやすい
- 理由 2: easy-agent.agent.md の Fallback Chain は「上位 orchestration ロジック」（どのフェーズに戻すか）を記述するもので、ステータスごとの細かい意味づけは SKILL.md が担う

両方を持つ理由は冗長性ではなく**視点の違い**: SKILL.md は契約、easy-agent は戦略。意味は一致させるが、表現粒度は異なってよい。

## 4 サブエージェントの分類サマリ

ADR-009 制定時点での4スキルの構造:

| スキル | 階層 | 返却ステータス | Relay 適用 |
| :--- | :--- | :--- | :--- |
| `call-advisor` | 単一 | PROCEED / CORRECT / ESCALATE / STOP | (該当なし: フォールバックなし) |
| `call-refine-loop` | 単一 | CONVERGED / MAX_ITER / ESCALATE / ABORT(2種) | (該当なし: ユーザー転送ステータスは MAX_ITER のみ) |
| `call-parliament` | 2層 (議題 + 集約) | per-topic: AGREED / CONVERGED / MAX_ROUNDS / aggregate: 全 APPROVED / max_rejections / ERROR | ✅ max_rejections 超過時 |
| `call-hierarchy` | 2層 (タスク + 集約) | per-task: IN_REVIEW / ERROR / aggregate: 全 APPROVED / max_rejections / ERROR 連鎖 | ✅ max_rejections 超過時 |

## 設計の根拠

### なぜ ADR で「規約」として固定するのか

ADR-008 で導入されたパターンは強力だが、形式知化されていなかったため、後続の `call-parliament` / `call-hierarchy` / `call-advisor` への展開時に「2層にすべきか単一階層か」「Relay Principle をどう書くか」を毎回判断する必要があった。ADR-009 でこれを規約化することで:

1. 新規スキル追加時の設計負担を削減
2. 既存スキルの構造変更時に「規約から外れた」ことを検知可能
3. レビュアーが規約遵守を機械的にチェックできる

### なぜ「重複削除」ではなく「SSoT ルール」を選択したか

検討した代替案:
- (A) `easy-agent.agent.md` の Fallback Chain を完全に削除し、各 SKILL.md への参照のみにする
- (B) 逆に SKILL.md の Caller Response Contract を削除し、easy-agent.agent.md に集約する
- (C) **両方を維持し、SSoT ルールを規定する** ← 採用

(A) は easy-agent エージェントが orchestration 戦略を理解する文脈が失われ、エージェント定義としての可読性が低下する。(B) はパッケージ単位の自律性（SKILL.md がサブエージェントの完全な仕様を持つ）が崩れる。(C) は冗長性を許容するが、視点の違い（戦略 vs 契約）を活かす。

ドリフト検出の機械化は今後の課題として残す。

## 結果として得られる特性

- **一貫性**: 全サブエージェントスキルが同じセクション構造を持つ
- **ドリフト耐性**: SSoT ルールにより、ドキュメント間の矛盾発生時の解決手順が明確
- **拡張性**: 新規スキル追加時のテンプレートが規約として存在する
- **ユーザー透明性**: Relay Principle により、サブエージェントの状態がそのままユーザーに見える

## 今後の課題

- ~~規約遵守を CI で機械チェックする仕組み（SKILL.md に必須セクションが存在するかのリンター）~~ → **解決済み (2026-04-27)**: `.github/workflows/check.yml` に `Lint Caller Response Contract (ADR-009 rule 1)` ステップを追加。すべての `*/.apm/skills/call-*/SKILL.md` に「Caller Response Contract」または「呼び出し元の応答コントラクト」見出しが存在することを検証する。あわせて関連する以下の2リンタも追加:
  - `Lint ADR index completeness`: 全 ADR-XXX.md が `docs/adr/README.md` から参照されていることを検証 (ADR-008 が長期間未登録だった回帰の予防)
  - `Lint sister schema common base`: parliament/taskforce の `orchestrator_state.json` が共通ベース status enum (TODO/IN_PROGRESS/IN_REVIEW/APPROVED/REJECTED/ERROR) を持つことを検証 (拡張は許容)
- `easy-agent.agent.md` の Fallback Chain と各 SKILL.md の Caller Response Contract のドリフト検出（テキスト類似度ベースのチェッカ等）
- 新規 `call-X` スキル追加時のテンプレート / スキャフォールド

## 関連決定

- ADR-008: サブエージェント返却プロトコルの形式化 — 本 ADR の前身。`call-refine-loop` のみで導入されたパターンを4スキル全体に拡張するのが本 ADR の役割。
- ADR-006: Phase Gate Protocol — easy-agent.agent.md の Fallback Chain は本プロトコルの一部であり、SSoT ルール（規約 4）の対象。
