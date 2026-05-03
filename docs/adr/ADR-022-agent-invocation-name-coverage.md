# ADR-022: Agent Invocation Name Coverage (全呼び出しパターンのエージェント名検証)

## ステータス

Accepted

## コンテキスト

easy-agents のエージェントは環境に応じて3種類の呼び出しパターンを使い分ける。

| 環境 | 呼び出し形式 | 例 |
| :--- | :--- | :--- |
| VS Code / GitHub Copilot | `runSubagent(agentName: "X")` | `runSubagent(agentName: "memoir", ...)` |
| Claude Code | `agent(subagent_type: "X")` | `agent(subagent_type: "refine-loop", ...)` |
| Copilot CLI | `task(agent_type: "my-copilot:X")` | `task(agent_type: "my-copilot:advisor", ...)` |

ADR-016 がメモワーブリッジパターンを導入した際に「VS Code の `runSubagent(agentName: "X")` 参照が未定義エージェントを指していないか」を検証する CI lint が追加された。しかし、残り 2 パターン（Claude Code の `subagent_type` と Copilot CLI の `my-copilot:`）は未保護だった。

これにより以下のサイレント劣化が検出できなかった：

1. `.apm/skills/call-*/SKILL.md` や `.apm/agents/*.agent.md` 内の `subagent_type: "X"` がエージェントのリネーム後も旧名称のまま残る
2. `task(agent_type: "my-copilot:X")` の `X` が typo されても CI が気づかない
3. エージェントが削除された際に参照元が自動検出されない

## 決定

### 三パターン完全カバレッジ

`check.yml` に以下の 2 つの lint ステップを追加し、全呼び出しパターンをカバーする。

#### Lint I: `agent(subagent_type: "X")` — Claude Code パターン

`.apm/**/*.md` ファイル内の `subagent_type: "X"` 値が、いずれかのパッケージの `.apm/agents/*.agent.md` フロントマター `name:` フィールドに存在することを確認する。

```python
SUBAGENT_TYPE_RE = re.compile(r'subagent_type:\s*["\']([A-Za-z0-9_-]+)["\']')
```

#### Lint J: `task(agent_type: "my-copilot:X")` — Copilot CLI パターン

`.apm/**/*.md` ファイル内の `agent_type: "my-copilot:X"` の `X` 部分が、いずれかのエージェントの `name:` に存在することを確認する。

```python
COPILOT_RE = re.compile(r'agent_type:\s*["\']my-copilot:([A-Za-z0-9_-]+)["\']')
```

### テンプレートプレースホルダーの除外

両 lint とも正規表現の文字クラス `[A-Za-z0-9_-]+` が `{` にマッチしないため、
`"{agent_name}"` や `"my-copilot:{agent-name}"` などのプレースホルダーは自動的に除外される。
明示的な PLACEHOLDER_RE チェックは不要。

### スキャン対象の統一

3 つの lint すべて `*/.apm/**/*.md` のみをスキャン対象とする（README や docs は対象外）。
実際に実行されうるソースファイルのみを保護し、ドキュメント内の説明例で誤検知が起きないようにする。

## 設計の根拠

### なぜパターンごとに別 lint ステップか

各パターンは独立した正規表現を持ち、エラーメッセージも異なる（`runSubagent(agentName: ...)` vs
`agent(subagent_type: ...)` vs `task(agent_type: my-copilot:...)`）。同一ステップにまとめると
エラーの発見箇所が曖昧になるため、分割して明確な診断を提供する。

### なぜ README はスキャン対象外か

README 内のコードブロックはドキュメント用の例示であり、実行されない。既存の `runSubagent` lint
も同じ方針（`*/.apm/**/*.md` のみスキャン）を採用しており、一貫性を維持する。

## 結果

| パターン | lint ステップ | 追加前の状態 | 追加後の状態 |
| :--- | :--- | :--- | :--- |
| VS Code `runSubagent(agentName)` | 既存 lint（ADR-016 由来） | ✅ 保護済み | ✅ 保護済み |
| Claude Code `agent(subagent_type)` | Lint I (本 ADR) | ❌ 未保護 | ✅ 保護済み |
| Copilot CLI `task(agent_type: my-copilot:)` | Lint J (本 ADR) | ❌ 未保護 | ✅ 保護済み |

現在スキャン対象 `.apm/**/*.md` 内で検出される全参照（Lint I: 4件、Lint J: 8件）は
すべて有効なエージェント名を指しており、lint は正常に pass する。

## 今後の課題

- `.apm/**/*.md` 以外のソースファイル（`.py`、`.ts` 等）でエージェント名を参照する場合、
  本 ADR の範囲外となる。必要に応じて lint を拡張する。
- エージェント名の変更時は、すべての参照を一括更新するためのリネームスクリプト整備を検討できる。

## 関連 ADR

- [ADR-015](./ADR-015-dispatch-failure-protocol.md) — Dispatch Failure Protocol（エージェント起動失敗の正規処理）
- [ADR-016](./ADR-016-memoir-agent-bridge.md) — Memoir Agent Bridge Pattern（runSubagent lint の原点）
- [ADR-017](./ADR-017-symmetric-output-schema-coverage.md) — Symmetric Output Schema Coverage（全スキルへの対称的保護の先例）
