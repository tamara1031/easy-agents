# ディレクトリ構成

## リポジトリルート

```
easy-agents-hub/
├── .claude/
│   └── skills/
│       └── empirical-prompt-tuning/   # APMで管理される共有スキル
├── apm_modules/                        # APM依存モジュール (git-ignored)
├── apm.yml                             # ワークスペース APM設定
├── apm.lock.yaml                       # 依存ロックファイル
├── .gitmodules                         # サブモジュール定義
├── docs/                               # 本ドキュメント群
│   ├── README.md
│   ├── directory-structure.md
│   ├── agents.md
│   ├── skills.md
│   ├── dependencies.md
│   └── adr/
├── easy-agent/                         # [submodule] コアフレームワーク
├── advisor/                            # [submodule] アドバイザー
├── parliament/                         # [submodule] 合意形成
├── taskforce/                          # [submodule] タスク実行
└── memoir/                             # [submodule] 長期記憶
```

## 各サブモジュールの構成パターン

全モジュールは共通の APM パッケージ構造に従います。

```
<module>/
├── MANIFEST.json          # エージェント・スキル宣言
├── plugin.json            # Claude プラグインメタデータ
├── apm.yml                # APMパッケージ設定・依存関係
├── apm.lock.yaml          # 依存バージョンロック
├── README.md              # (現状空)
├── agents/                # エージェント定義 (*.agent.md)
└── skills/
    └── <skill-name>/
        ├── SKILL.md       # スキル本体ドキュメント
        ├── schemas/       # JSON スキーマ (入出力契約)
        ├── templates/     # テンプレートファイル
        ├── references/    # 参考・補助ドキュメント
        └── scripts/       # 実行スクリプト (memoir のみ)
```

## モジュール別ファイルツリー

### easy-agent
```
easy-agent/
├── agents/
│   └── easy-agent.agent.md         # 統合オーケストレーター定義
├── .github/skills/
│   └── empirical-prompt-tuning/    # (git-ignored)
├── MANIFEST.json
├── plugin.json
├── apm.yml
└── apm.lock.yaml
```

### advisor
```
advisor/
├── agents/
│   └── advisor.agent.md            # アドバイザーエージェント定義
├── skills/
│   └── call-advisor/
│       ├── SKILL.md                # call-advisorスキル
│       └── references/
│           └── prompt-template.md  # 相談プロンプト例5件
├── MANIFEST.json
├── plugin.json
├── apm.yml                         # 依存: easy-agent, memoir, parliament, taskforce
└── apm.lock.yaml
```

### parliament
```
parliament/
├── agents/
│   ├── parliament-chairperson.agent.md  # 議長エージェント
│   └── parliament-member.agent.md       # 議員エージェント
├── skills/
│   └── call-parliament/
│       ├── SKILL.md
│       ├── schemas/
│       │   ├── member_message.json      # 議員発言フォーマット
│       │   ├── chairperson_output.json  # 議長出力フォーマット
│       │   └── orchestrator_state.json  # 状態管理スキーマ
│       └── templates/
│           └── stance_definitions.md    # スタンス定義
├── MANIFEST.json
├── plugin.json
└── apm.yml
```

### taskforce
```
taskforce/
├── agents/
│   ├── hierarchy-manager.agent.md   # 階層マネージャー
│   └── hierarchy-member.agent.md    # 階層メンバー
├── skills/
│   └── call-hierarchy/
│       ├── SKILL.md
│       ├── schemas/
│       │   ├── manager_output.json
│       │   ├── member_output.json
│       │   └── orchestrator_state.json
│       └── templates/
│           ├── checklist.md
│           └── status_definitions.md
├── MANIFEST.json
├── plugin.json
└── apm.yml
```

### memoir
```
memoir/
├── skills/
│   └── long-term-memory/
│       ├── SKILL.md                    # スキル定義 (324行・日本語)
│       ├── docker/
│       │   └── docker-compose.yml      # ChromaDB コンテナ
│       └── scripts/
│           ├── _common.py              # 共通ユーティリティ
│           ├── memory_save.py          # 保存
│           ├── memory_search.py        # 検索
│           ├── memory_update.py        # 更新
│           └── memory_delete.py        # 削除
├── MANIFEST.json
├── plugin.json
└── apm.yml
```
