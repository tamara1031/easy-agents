# Auto-Memory Protocol (自動記憶保存プロトコル)

> **Canonical Source** for `[role: hook]` — `easy-agent/.apm/hooks/session-start.md`
> `easy-agent.agent.md` の "Auto-Memory Protocol" セクションは本ファイルの Assembled View。
> 編集は必ず本ファイルを正として行い、agent.md へ反映すること。

会話から得た情報を memoir の **`long-term-memory` スキル**経由で ChromaDB に保存し、将来のセッションでユーザーの役割・好み・プロジェクト状況をすぐに把握できるようにする。スクリプトへの直接呼び出しは行わず、常にスキルのインターフェースを通じて操作する。

## 発火トリガー（いつ保存するか）

記憶タイプにより **即座保存** と **フェーズゲート保存** の 2 つのタイミング体系がある。

| 記憶タイプ | 発火タイミング | 体系 | memoir タグ |
| :--- | :--- | :--- | :--- |
| `user` | ユーザーの役割・スキルレベル・経験事実が初めて言及された exchange | **即座** | `user` |
| `user-pref` | ユーザーの行動傾向・好み・将来の意向が初めて言及された exchange | **即座** | `user-pref` |
| `feedback` | 条件成立後、**最寄りの Phase Gate で verdict が APPROVED のとき**に保存。REVISE / LOOPBACK / DELEGATE / ESCALATE の場合はスキップし、次の APPROVED gate まで持ち越す | **フェーズゲート** | `feedback`, `rule` |
| `project` | **Phase Gate の verdict が APPROVED** で、かつフェーズ完了時に直前の `project` 記憶と比べて TaskScale・変更対象ファイルリスト・フェーズ状態・主要成果物（Explore: 調査レポート / Plan: チェックリスト / Implement: 変更ファイルリスト / Verify: テスト結果）のいずれかが変化したとき | **フェーズゲート** | `project`, `project-rule` |
| `reference` | 外部システムの URL・ボード・チャンネルが言及された exchange | **即座** | `reference` |

> **即座保存は Phase Gate プロトコルの制約対象外**。`user` / `reference` はフェーズ完了を待たず、その exchange で書き込む。
>
> **タグ使い分けの補足**: `user` = ユーザーの現在の役割・スキルレベル・経験事実。`user-pref` = ユーザーの行動傾向・好み・将来の意向（特定プロジェクト成果物に紐付かない）。`project` = 具体的な成果物（ファイルリスト・フェーズ状態・変更対象）の変化と紐付く情報のみ。

## サイレント承認の閾値

「非デフォルト選択の黙認」＝別のエージェントが合理的に異なる判断をするところを、ユーザーが訂正なく通過させた場合。  
**同一パターンの 2回確認** で `feedback` 発火。明示的修正は **1回** で発火。

## 最小保存ルール

（1 exchange ＝ ユーザーメッセージ 1件 ＋ エージェント応答 1件）

前回保存から **15 exchange 以上**が経過している場合、次のフェーズへ進む前（またはセッション終了時）に有機的な学習を確認する。保存対象が存在する場合のみ保存し、空または低価値エントリの強制保存は行わない。

## 保存手順

> **環境別の対応**:
> - **Claude Code**: `Skill` ツールで `long-term-memory` スキルを呼び出す（以下のフォーマット参照）。
> - **VS Code / GitHub Copilot**: `Skill` ツールは利用できない。memoir エージェントを `runSubagent` で呼び出すか、長期記憶保存をスキップして Copilot 標準のコンテキスト管理に委ねること。
>   ```javascript
>   runSubagent(agentName: "memoir", description: "Save memory", prompt: "Save: items=[{text: '<本文>', tags: ['<memoir タグ>']}], source='session', dedup=true")
>   ```
> - **memoir 利用不可時（`Skill` ツール不可・Docker 未起動・スクリプトエラーなど）**: memoir オペレーション全体を**静かにスキップ**し、そのまま通常の処理を続行する。「ファイル操作ツールなし」「環境制限」「前回と同様に直接作業」等の文言を一切出力しないこと。memoir の失敗は `read`/`edit`/`search` ツールの可用性とは無関係であり、ファイル操作ツールは memoir とは独立して利用可能。

Claude Code の **Skill ツール**で `long-term-memory` スキルを呼び出す（Python スクリプトを直接 Bash 実行しない）。知識を 1 ファクト＝1 ナレッジ単位に分解し、`dedup` を有効にして保存する。

```
Skill ツール呼び出し:
  skill: "long-term-memory"
  args: (Save オペレーション)
    items: [{"text": "<本文>", "tags": ["<memoir タグ>"]}]
    source: "session"
    dedup: true
```

- **items.text**: 文脈情報を含め単独で意味が通じるテキスト
- **items.tags**: 発火トリガー表の「memoir タグ」列を参照
- **dedup**: 必須（記憶の肥大化を防止）

**`feedback` / `project` テキスト構造**: 「ルール/事実」→「Why: 理由」→「How to apply: 適用基準」の順で 1 ユニットにまとめる。

## project 型の変化検出

> **VS Code / GitHub Copilot**: Skill ツールが使えない場合は `runSubagent` で Search を行うか、変化検出をスキップして無条件に保存を実行する。
> ```javascript
> runSubagent(agentName: "memoir", description: "Search project memory", prompt: "Search: query='project phase state artifacts', tags='project', n-results=1")
> ```

保存前に Skill ツールで Search オペレーションを呼び出し、直前の `project` 記憶と比較する（Search の tags はカンマ区切り文字列、Save の items.tags は配列 — memoir の CLI 仕様に準拠）：

```
Skill ツール呼び出し:
  skill: "long-term-memory"
  args: (Search オペレーション)
    query: "project phase state artifacts"
    tags: "project"
    n-results: 1
```

score ≥ 0.60 の結果と TaskScale・変更対象ファイルリスト・フェーズ状態・主要成果物を比較し、1つ以上変化していれば新規保存。変化なしなら保存スキップ。（score は cosine similarity: 0〜1、高いほど類似）

## セッション開始時のリコール

> **VS Code / GitHub Copilot**: Skill ツールが使えない場合は `runSubagent` で Search を行うか、リコールをスキップして Copilot 標準のコンテキスト管理に委ねる。
> ```javascript
> runSubagent(agentName: "memoir", description: "Recall session context", prompt: "Search: query='user preferences project context feedback rules', n-results=10")
> ```

セッション開始時に Skill ツールで Search を呼び出し、コンテキストを復元する：

```
Skill ツール呼び出し:
  skill: "long-term-memory"
  args: (Search オペレーション)
    query: "user preferences project context feedback rules"
    n-results: 10
```

score ≥ 0.60 の結果を踏まえて応答する。（score は cosine similarity: 0〜1、高いほど類似）qualifying results が 0 件の場合は「前セッションの記憶が見つかりませんでした。現在の作業コンテキストを共有してください。」と告知してから回答する。

memoir スキルの呼び出し自体が失敗した場合（`Skill` ツール不可・Docker 未起動・スクリプトエラー等）: エラー文言を出力せず、通常のセッション開始として扱いユーザーのタスクに直接応答する。「前回と同様に」「ファイル操作ツールなし」「環境制限」「As-Is Report を確認」等のフレーズは**絶対に使用しない**。

## Hook イベント語彙

| イベント名 | 発火条件 | 本ファイルの対応処理 |
| :--- | :--- | :--- |
| `SessionStart` | エージェントが新しいセッションを開始したとき | memoir からのリコール実行 |
| `PhaseGateComplete` | Phase Gate の verdict が APPROVED のとき | `feedback` / `project` タイプの保存評価 |
| `NExchangeElapsed(15)` | 前回保存から 15 exchange 経過したとき | 最小保存ルールのチェック |
| `ImmediateTrigger` | `user` / `user-pref` / `reference` タイプの情報が言及されたとき | 即座保存実行 |
