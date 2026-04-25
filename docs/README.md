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
   └─→ memoir         : 長期記憶 (ChromaDB)
```

## モジュール一覧

| モジュール | バージョン | 役割 |
|---|---|---|
| easy-agent | 0.1.0 | ユニバーサルオーケストレーター |
| advisor | 0.1.0 | 戦略的アドバイザー (Opus) |
| parliament | 0.1.0 | 合意形成・設計討議 |
| taskforce | 0.1.0 | 階層型タスク実行 |
| memoir | 0.1.0 | ベクターDB長期記憶 |
