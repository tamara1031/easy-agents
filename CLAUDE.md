# easy-agents プロジェクト CLAUDE.md

## プロジェクト概要

このリポジトリは Claude Code 向けのマルチエージェントフレームワーク群を APM パッケージとして管理する。パッケージは `easy-agent`, `advisor`, `parliament`, `taskforce`, `memoir` の5つ。

## APM パッケージの仕組み

APM (Agent Package Manager) における役割:

| ディレクトリ | 役割 | git 追跡 |
|---|---|---|
| `.apm/` | **ソース** — エージェント・スキルをここに書く | 追跡する |
| `.github/` | ビルド成果物 — `apm install` が生成 | 無視する |
| `.claude/` | ビルド成果物 — `apm install` が生成 | 無視する |
| `apm_modules/` | インストール済み依存パッケージ | 無視する |

`.apm/` は npm の `src/`、`.github/` と `.claude/` は `dist/` に相当する。

## パッケージ構成

各サブパッケージは以下の構造を持つ:

```
<package>/
├── plugin.json            # Claude プラグインメタデータ
├── apm.yml                # APM パッケージ設定
├── README.md
├── .gitignore             # .claude/ .github/ apm_modules/ を除外
└── .apm/                  # ソースディレクトリ (git 追跡対象)
    ├── agents/            # エージェント定義 (*.agent.md)
    └── skills/            # スキル定義 (SKILL.md + 付属ファイル)
```

`agents/`・`skills/` のルートレベルディレクトリは存在しない。`.apm/` が唯一のソース。

## エージェント・スキルを変更した場合

```bash
# 1. .apm/ 配下のファイルを直接編集
vi <package>/.apm/agents/*.agent.md
vi <package>/.apm/skills/<skill-name>/SKILL.md

# 2. 変更をコミット
git add <package>/.apm/
git commit -m "feat(<package>): ..."

# 3. ルートプロジェクトで apm install を実行してデプロイ
apm install --no-policy --update
```

`apm install` が `.apm/` を読み取り `.github/` と `.claude/` を生成する。sync スクリプトは不要。

## 新しいパッケージを追加する場合

```bash
# 1. ディレクトリ骨格を作成
mkdir -p <package>/.apm/{agents,skills/<skill-name>}

# 2. apm.yml を作成
cat > <package>/apm.yml << 'EOF'
name: <package>
version: 0.1.0
description: ...
author: tamara1031
license: MIT
target: [claude, copilot]
type: hybrid
dependencies:
  apm: []
  mcp: []
scripts: {}
EOF

# 3. .gitignore を作成
cat > <package>/.gitignore << 'EOF'
.claude/
.github/

# APM dependencies
apm_modules/
EOF

# 4. .apm/agents/*.agent.md と .apm/skills/**/SKILL.md を作成

# 5. コミット
git add <package>/
git commit -m "feat: add <package> package"
```

## root プロジェクトでの apm install

```bash
# ローカルキャッシュから (.apm/ が commit 済みの場合)
apm install --no-policy

# GitHub から最新を再取得してインストール
apm install --no-policy --update
```

インストール後に生成されるファイル:
- `.claude/agents/*.md` — サブエージェント定義
- `.claude/skills/*/SKILL.md` — スキル定義
- `.github/agents/*.agent.md` — GitHub Copilot 向けエージェント定義

## 依存関係

```
easy-agent
├── advisor      (Opus アドバイザー)
├── parliament   (議会モデル)
├── taskforce    (階層型タスク実行)
└── memoir       (ベクター長期記憶)
```

ルートの `apm.yml` は `easy-agent` のみを直接依存として宣言し、残りは推移的依存として解決される。

## コードスタイル

- エージェント定義 (`*.agent.md`): YAML frontmatter + Markdown
- スキル定義 (`SKILL.md`): Markdown のみ
- `apm.yml` の `type` フィールドは必ず `hybrid` にする
