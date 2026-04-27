# easy-agents-hub ドキュメント

easy-agents-hub は、Claude Code 上で動作する複数の専門エージェント・スキルをまとめたモノレポです。

## ドキュメント構成

| ファイル | 内容 |
|---|---|
| [directory-structure.md](./directory-structure.md) | リポジトリ全体のディレクトリ構成 |
| [agents.md](./agents.md) | エージェント定義と役割 |
| [skills.md](./skills.md) | スキル定義と使い方 |
| [dependencies.md](./dependencies.md) | モジュール間依存関係 |
| [adr/](./adr/README.md) | Architecture Decision Records |

## システム概要

```
ユーザー
   ↓
easy-agent (統合オーケストレーター)
   ├─→ advisor        : 戦略的判断 (Opus)
   ├─→ parliament     : 設計コンセンサス (多視点討議)
   ├─→ taskforce      : 実装実行 (Plan→Implement→Review)
   ├─→ refine-loop    : Verify フェーズの反復改善 (バイアスフリーレビュー)
   └─→ memoir         : 長期記憶 (ChromaDB)
```

## モジュール一覧

| モジュール | バージョン | 役割 |
|---|---|---|
| easy-agent | 0.1.0 | ユニバーサルオーケストレーター |
| advisor | 0.1.0 | 戦略的アドバイザー (Opus) |
| parliament | 0.1.0 | 合意形成・設計討議 |
| taskforce | 0.1.0 | 階層型タスク実行 |
| refine-loop | 0.1.0 | 反復改善ループ (Verify フェーズ) |
| memoir | 0.1.0 | ベクターDB長期記憶 |

## 新規 call-X スキルの追加

新しいサブエージェントスキルを追加する場合は、以下の手順に従う ([ADR-009](./adr/ADR-009-caller-response-contract-convention.md) 規約 1〜4 への準拠が必須):

1. **スキャフォールドをコピー**:
   ```bash
   cp docs/templates/call-skill-template.md \
      <package>/.apm/skills/call-<skill-name>/SKILL.md
   ```
2. **テンプレート内の `{placeholder}` と `{# コメント}` を実装内容で置換**。Caller Response Contract セクションは削除しないこと（CI で必須セクションを機械チェック）。
3. **2層 vs 単一階層の選択**: サブエージェントが複数処理単位（議題・タスク等）を持つ場合は Pattern B（per-unit + orchestrator-aggregate）、そうでない場合は Pattern A（単一階層）を採用。判定基準は ADR-009 規約 2 を参照。
4. **対応エージェント定義を作成**: `<package>/.apm/agents/<agent-name>.agent.md` を作成し、frontmatter (name / description / model / user-invocable / tools) を設定。
5. **ナビゲーションドキュメントを更新**:
   - `docs/skills.md` のスキル一覧に1行追加
   - `docs/agents.md` のエージェント一覧に1行追加
   - `docs/dependencies.md` の呼び出し関係に追記（必要に応じて）
6. **CI 検証**:
   - `Lint Caller Response Contract` ステップが新スキルを発見し、必須見出しの存在を確認
   - `apm install` が成功し `.claude/skills/` にビルド成果物が生成される
7. **必要に応じて ADR を追加**: 新パターンが既存 ADR と矛盾する場合は新規 ADR で記録し、`docs/adr/README.md` の索引にも追加（CI の `Lint ADR index completeness` がカバー）。
