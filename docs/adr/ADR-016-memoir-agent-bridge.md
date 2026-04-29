# ADR-016 — Memoir Agent Bridge Pattern (VS Code 長期記憶アクセス)

## ステータス

Accepted

## 背景

memoir パッケージは `long-term-memory` スキルを提供し、Claude Code 環境では `Skill` ツールで直接呼び出せる。しかし VS Code / GitHub Copilot 環境では `Skill` ツールが利用できないため、エージェントが長期記憶を利用する手段が存在しなかった。

`easy-agent` の Auto-Memory Protocol（`easy-agent/.apm/hooks/session-start.md`）は VS Code 向けに `runSubagent(agentName: "memoir", ...)` というフォールバックパスを提供していた。しかし、`memoir` という名前のエージェント定義（`memoir.agent.md`）が存在せず、このパスは**定義なし参照**となっていた。

この状況を CI が検知できなかったことも問題であった（ADR-016 以前は `runSubagent` 参照と agent 定義の対応関係を lint するルールが存在しなかった）。

## 決定

### 1. `memoir.agent.md` の追加

`memoir/.apm/agents/memoir.agent.md` を新設し、VS Code 環境での長期記憶ブリッジとして機能させる。

**設計方針**:
- 受け取ったプロンプト（`Save:` / `Search:` / `Update:` / `Delete:` コマンド形式）を memoir スクリプト（`memory_save.py` 等）の CLI 呼び出しに変換する
- ツールは `execute` のみ使用（ファイル編集・検索・エージェント起動は不要）
- エラー時は JSON 形式で返却し、呼び出し元が静かにスキップできるようにする
- `user-invocable: false`（Auto-Memory Protocol が自動的に呼び出す）

### 2. CI Lint の追加（Lint runSubagent agent name references）

`check.yml` に新しい lint ステップを追加し、`.apm/**/*.md` ファイル内の全 `runSubagent(agentName: "X", ...)` 呼び出しに対して、対応する `X.agent.md` が存在することを検証する。テンプレートプレースホルダー（`{agent-name}`）は除外する。

## 環境別の memoir アクセスパターン

| 環境 | アクセス方法 | ファイル |
| :--- | :--- | :--- |
| Claude Code | `Skill` ツール (`long-term-memory`) | `memoir/.apm/skills/long-term-memory/SKILL.md` |
| VS Code / GitHub Copilot | `runSubagent(agentName: "memoir", ...)` | `memoir/.apm/agents/memoir.agent.md` |
| memoir 利用不可時 | 静かにスキップ（Degrade-and-Continue） | ADR-015 相当 |

## 否定した代替案

### A. VS Code 向け memoir を `long-term-memory` スキル内で分岐

`SKILL.md` に VS Code 向けコードブロックを追加することを検討した。しかし `SKILL.md` は Claude Code の `Skill` ツールが解釈するものであり、VS Code では `runSubagent` の対象となるエージェント定義が必要。スキルにエージェントロジックを混在させることは ADR-010 の役割分離に反する。

### B. memoir エージェントを user-invocable にする

Claude Code では既に `long-term-memory` スキルが user-invocable。memoir エージェントは VS Code の自動記憶保存 hook から呼ばれるものであり、ユーザーが直接起動する必要はない。`user-invocable: false` が適切。

## 影響範囲

| ファイル | 変更内容 |
| :--- | :--- |
| `memoir/.apm/agents/memoir.agent.md` | **新規追加** — VS Code ブリッジエージェント |
| `.github/workflows/check.yml` | **新規 lint 追加** — runSubagent 参照の存在検証 |
| `docs/agents.md` | memoir エージェントのエントリを一覧・説明に追加 |

## 関連 ADR

- [ADR-004](./ADR-004-vector-memory.md) — ベクターDB長期記憶 (ChromaDB)
- [ADR-011](./ADR-011-hook-specification-format.md) — Hook Specification Format（Auto-Memory Protocol の hook イベント）
- [ADR-015](./ADR-015-dispatch-failure-protocol.md) — Dispatch Failure Protocol（memoir 利用不可時のスキップ動作）
