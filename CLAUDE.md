# easy-agents プロジェクト CLAUDE.md

## プロジェクト概要

このリポジトリは Claude Code 向けのマルチエージェントフレームワーク群を APM パッケージとして管理する。パッケージは `easy-agent`, `advisor`, `parliament`, `taskforce`, `memoir` の5つ。

## パッケージ構成

各サブパッケージは以下の構造を持つ:

```
<package>/
├── plugin.json            # Claude プラグインメタデータ
├── apm.yml                # APM パッケージ設定 (type: hybrid, scripts.sync 必須)
├── README.md
├── .gitignore
├── .apm/                  # デプロイ用コンパイル成果物 (git 追跡対象)
│   ├── .gitkeep
│   ├── agents/            # apm run sync で agents/ から同期
│   └── skills/            # apm run sync で skills/ から同期
├── agents/                # エージェント定義ソース (*.agent.md)
└── skills/                # スキル定義ソース
```

## 重要: .apm/ ディレクトリの管理

`.apm/` は `apm install` が依存パッケージのリソースをデプロイする際に参照するディレクトリ。`.apm/` にファイルがないと、パッケージはインストール済みとして認識されるが何もデプロイされない。

**`.apm/` は `apm install` では自動生成されない。`apm run sync` で手動同期してからコミットする。**

### エージェント・スキルを変更した場合

```bash
# 1. ソースファイルを編集
vi <package>/agents/*.agent.md  # または skills/**/SKILL.md

# 2. .apm/ に同期 (packages/ ディレクトリ内で実行)
cd <package>/
apm run sync
# → .apm/agents/ と .apm/skills/ が agents/ と skills/ から上書き同期される

# 3. 変更ファイルをすべてコミット (ソース + .apm/ の両方)
git add agents/ skills/ .apm/
git commit -m "feat(<package>): ..."
```

### 新しいパッケージを追加する場合

```bash
# 1. ディレクトリ骨格を作成
mkdir -p <package>/{agents,skills/<skill-name>,.apm}
touch <package>/.apm/.gitkeep

# 2. apm.yml を作成 (type: hybrid と scripts.sync は必須)
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
scripts:
  sync: "rm -rf .apm/agents .apm/skills && cp -r agents .apm/ && cp -r skills .apm/"
EOF
# ※ agents/ のみのパッケージは skills を除く、skills/ のみは agents を除く

# 3. plugin.json を作成 (advisor パターンを参考に)

# 4. agents/*.agent.md と skills/**/SKILL.md を作成

# 5. .apm/ に同期してコミット
apm run sync
git add .
git commit -m "feat: add <package> package"
```

### .gitignore テンプレート

新規パッケージには以下の `.gitignore` を作成する:

```
.claude/
.github/skills/

# APM dependencies
apm_modules/
```

## root プロジェクトでの apm install

```bash
# ローカルキャッシュから (各パッケージの .apm/ を commit/push 済みの場合)
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
