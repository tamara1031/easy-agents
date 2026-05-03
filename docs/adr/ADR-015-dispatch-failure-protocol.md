# ADR-015: Dispatch Failure Protocol — サブエージェント起動失敗の正規処理規約

- **ステータス**: Accepted
- **決定日**: 2026-04-28

## コンテキスト

ADR-009 は全 `call-*/SKILL.md` に「Caller Response Contract」を義務付け、各スキルが返却しうるステータスとその意味・呼び出し元のアクションを網羅することを要求した。しかし既存の Caller Response Contract が対象としているのは **「サブエージェントが起動に成功し、何らかのステータスを返した」場合** のみであり、**サブエージェント自体が起動できなかった、またはタイムアウトした場合（dispatch failure）** の処理が未定義のままである。

### 現状の不対称性

| スキル | dispatch failure の扱い |
| :--- | :--- |
| `call-refine-loop` | `ABORT (dispatch 不可)` として Caller Response Contract に記述済み ✅ |
| `call-parliament` | 未定義 — 議長の起動失敗に関するコントラクトなし ❌ |
| `call-hierarchy` | 未定義 — マネージャーの起動失敗に関するコントラクトなし ❌ |
| `call-advisor` | 未定義 — `フォーマット判定` で `PROCEED` フォールバックのみ存在 (dispatch failure とは別問題) ❌ |

この非対称性は ADR-009 のカバレッジの盲点であり、以下の問題を引き起こす。

1. **呼び出し元 (easy-agent) の振る舞いが未定義**: `agent` ツール不可・タイムアウト時に easy-agent がどう振る舞うべきかの指針がない。
2. **CI による検証不能**: dispatch failure ケースが SKILL.md に記述されていないため、Lint でカバレッジを検証できない。
3. **退行リスク**: 新スキル追加時に dispatch failure 処理が書かれないまま CI をパスしてしまう。

## 決定

### 1. `DISPATCH_FAILURE` を正規ステータスとして定義する

`DISPATCH_FAILURE` は **サブエージェントの dispatch メカニズム自体の失敗**（起動失敗・タイムアウト・`agent` / `task` / `runSubagent` ツール不可）を表す正規ステータスである。

| 概念 | 区別 |
| :--- | :--- |
| `DISPATCH_FAILURE` | サブエージェントが **起動できなかった / 応答を返さなかった**。呼び出しが達しなかった状態。 |
| サブエージェント返却 `ERROR` | サブエージェントが **起動し実行したが、内部で致命的失敗が発生**して ERROR を返した状態。 |

> `DISPATCH_FAILURE` は ADR-009 の返却ステータス体系の外側に位置する「インフラ層の失敗」であり、サブエージェント定義の `ERROR` とは意味的に独立している。

### 2. 全 `call-*/SKILL.md` に `DISPATCH_FAILURE` の処理を義務付ける

すべての `call-X/SKILL.md` の「呼び出し元の応答コントラクト (Caller Response Contract)」セクションは、`DISPATCH_FAILURE` に対する呼び出し元アクションを明示しなければならない。

### 3. 標準フォールバック分類

スキルの役割に応じて、以下の3分類から適切なフォールバックを選択する。

| 分類 | 適用スキル | フォールバック戦略 |
| :--- | :--- | :--- |
| **Degrade-and-Continue** | `call-advisor` | サブエージェントなしで現在のアプローチを自律継続。ユーザーに通知する。 |
| **Skip-and-Report** | `call-parliament`, `call-hierarchy` | 失敗した処理単位を ERROR 扱いでスキップし、残存単位の処理を継続する。全単位が失敗した場合は STOP。 |
| **Fallback-Mode** | `call-refine-loop` | 既存の `ABORT (dispatch 不可)` 処理と等価。内部ループ (REVISE) にフォールバックする。 |

### 4. CI による `DISPATCH_FAILURE` 宣言の検証

`check.yml` に以下の Lint ステップを追加する。

#### Lint F: `DISPATCH_FAILURE` 宣言の存在検証

全 `call-*/SKILL.md` が `DISPATCH_FAILURE` の文字列を含むことを検証する。Caller Response Contract セクション内での宣言を想定するが、CI はセクション位置を問わず文字列の存在のみを確認する。

### 5. CI ドリフト検出の拡張

既存の「Lint Fallback Chain vs Caller Response Contract」ステップを拡張し、以下を追加する。

- `call-advisor` を検証対象に追加（現状は対象外）
- 各スキルに `DISPATCH_FAILURE` がドリフト対象トークンとして追加

### 6. `docs/templates/call-skill-template.md` への反映

新規スキル作成用テンプレートに `DISPATCH_FAILURE` 行を必須エントリとして追加する。

## 設計の根拠

### なぜ新規ステータスとして定義するか

既存スキルには dispatch failure を暗黙的に扱う機構が各々存在する（`ABORT (dispatch 不可)` / `フォーマット判定の PROCEED フォールバック` など）。しかしこれらは:

1. 名称が統一されておらず、呼び出し元が統一的に処理できない
2. Caller Response Contract に記述されていないものもあり、契約外の動作となっている
3. CI で検証できない

`DISPATCH_FAILURE` を正規ステータスとして定義することで、ADR-009 の Caller Response Contract に「dispatch failure」ケースを正式に組み込み、CI による退行検出を可能にする。

### なぜ `ABORT` を再利用しないか

`call-refine-loop` の `ABORT` には2つの意味がある:

1. `ABORT ([critical] なし)`: requirements_checklist の設計上の問題
2. `ABORT (dispatch 不可)`: インフラ層の失敗

前者は「呼び出し元の使い方の誤り」、後者は「dispatch failure」であり、意味的に異なる。新たに `DISPATCH_FAILURE` を導入することで、`ABORT` の意味を「呼び出し元の設計ミス」に限定し、dispatch failure を明確に分離できる。既存の `ABORT (dispatch 不可)` 記述は後方互換性のため保持しつつ、`DISPATCH_FAILURE` との等価性を注記する。

### なぜ Degrade-and-Continue / Skip-and-Report / Fallback-Mode の3分類か

各スキルの役割から導かれる自然な分類である:

- **Advisor**: 1対1相談の補助役。起動できなくても主処理（エグゼキューターの実行）は独立して成立する → Degrade-and-Continue
- **Parliament / Hierarchy**: 処理単位（議題/タスク）の集合を管理する。1単位の失敗が残存単位をブロックすべきでない → Skip-and-Report
- **refine-loop**: 品質改善ループ全体がサブエージェントに依存している。dispatch failure = ループ不可 → Fallback-Mode (内部実行へ降格)

## 結果として得られる特性

- **Caller Response Contract の完全性**: 全スキルが「サブエージェント起動失敗」を含む全ケースをカバーする
- **CI による退行防止**: 新スキル追加時に `DISPATCH_FAILURE` 宣言漏れを CI が検出する
- **easy-agent の統一的処理**: Fallback Chain に `DISPATCH_FAILURE` ケースが追加され、オーケストレーターが統一的に応答できる
- **ADR-009 との対称性**: 成功パスと同様に、dispatch failure パスも「契約として明示・CI で強制」のサイクルに組み込まれる

## 今後の課題

- タイムアウト値の定義: dispatch failure を「タイムアウト」と「ツール不可」に細分化し、タイムアウト閾値を各スキルに明示する（現状は環境依存）
- `DISPATCH_FAILURE` の自動リトライポリシー: 一時的な起動失敗（ネットワーク瞬断等）と恒久的な起動失敗（ツール不可）を区別するリトライルールの定義
- `easy-agent/.apm/hooks/` への DISPATCH_FAILURE 処理の抽出: 将来的には hook として物理分離する（ADR-012 の方針に従い）

## 関連決定

- **ADR-009**: Caller Response Contract Convention — 本 ADR は ADR-009 のカバレッジを dispatch failure ケースに拡張する。
- **ADR-008**: サブエージェント返却プロトコルの形式化 — dispatch failure は「返却」が発生しないケースであり、ADR-008 の扱う範囲の外側を補完する。
- **ADR-011**: Hook Specification Format — DISPATCH_FAILURE 処理を将来的に hook として抽出する際の形式を規定する。
- **ADR-012**: 役割別物理ファイル分離 — 将来の hook 抽出時の物理ディレクトリ規約。
