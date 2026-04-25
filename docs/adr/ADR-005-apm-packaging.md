# ADR-005: APM によるパッケージ管理

- **ステータス**: Accepted
- **決定日**: 2026-04-25

## コンテキスト

複数のエージェント・スキルを複数のリポジトリに分散して管理する場合、バージョン管理・依存解決・デプロイの一貫性が課題となる。

## 決定

**APM (Agent Package Manager) v0.9.2** を全モジュールの依存管理に採用する。

### 標準ファイル構成
- `apm.yml`: パッケージ名・バージョン・依存関係宣言
- `apm.lock.yaml`: 依存の解決済みコミットハッシュを固定
- `MANIFEST.json`: モジュールが提供するエージェント・スキルの宣言
- `plugin.json`: Claude Code プラグインメタデータ

### コンパイル・インストール

```bash
apm install --no-policy --target copilot,claude
apm compile --target copilot,claude
```

## 結果

### メリット
- 各モジュールを独立した GitHub リポジトリとして管理 (サブモジュール構成)
- `apm.lock.yaml` でコミットハッシュ固定、再現性保証
- `empirical-prompt-tuning` 等の外部スキルを共有可能 (mizchi/skills)
- `.gitignore` で `apm_modules/` を除外し、リポジトリの軽量化

### デメリット・リスク
- APM v0.9.2 はプレリリース品質 (仕様変更の可能性)
- サブモジュールの detached HEAD 状態での操作に注意が必要
- `advisor/apm.yml` が固定コミットハッシュで依存を宣言しており、他モジュール更新時に手動更新が必要

## 注意事項

**モジュール間の依存バージョンの現状 (要更新)**:

`advisor/apm.yml` は以下の旧コミットに依存宣言している:
- easy-agent: `77294e6cb38b1d4e7748ae861f3182dbaf9812c6` (最新でない可能性)
- memoir: `0e56d880562ca00e4a163e8dc382f16467ed0031` (最新でない可能性)
- parliament: `6041d24dabe235070dc9566b302a0ab96a435be4` (最新でない可能性)
- taskforce: `f3a47f86710b0f930111423dd2b3b7cdf4e320b0` (最新でない可能性)

現在の各モジュールの HEAD コミットと突き合わせて更新が必要。
