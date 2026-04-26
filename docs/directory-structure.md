# ディレクトリ構成

## リポジトリルート

```
easy-agents-hub/
├── apm.yml                             # ワークスペース APM 設定
├── apm.lock.yaml                       # 依存ロックファイル
├── .gitignore                          # Git 管理除外設定
├── docs/                               # 本ドキュメント群
│   ├── README.md
│   ├── directory-structure.md
│   ├── agents.md
│   ├── skills.md
│   ├── dependencies.md
│   └── adr/                            # Architecture Decision Records
├── easy-agent/                         # コアオーケストレーター
├── advisor/                            # 戦略アドバイザー (Opus)
├── parliament/                         # 合意形成 (議会モデル)
├── taskforce/                          # 階層型タスク実行
└── memoir/                             # ベクター長期記憶
```

`apm install` 実行時にルートワークスペースが各サブモジュールを APM パッケージとして解決する。サブモジュールは git submodule ではなく単一リポジトリ内のディレクトリとして管理されている。

## 各サブパッケージの共通構成パターン

全パッケージは共通の APM パッケージ構造に従います。

```
<package>/
├── plugin.json            # Claude プラグインメタデータ (name/version/description/author/license)
├── apm.yml                # APM パッケージ設定・依存関係
├── README.md              # パッケージ概要・利用方法
├── .gitignore             # APM コンパイル成果物除外
├── .apm/                  # APM コンパイル成果物 (エージェント・スキルのデプロイ用コピー。git 追跡対象)
├── agents/                # エージェント定義 (*.agent.md)         ※ memoir には存在しない
└── skills/
    └── <skill-name>/
        ├── SKILL.md       # スキル本体ドキュメント
        ├── schemas/       # JSON スキーマ (入出力契約)            ※ parliament / taskforce のみ
        ├── templates/     # テンプレート / 定義集                 ※ parliament / taskforce のみ
        ├── references/    # 参考・補助ドキュメント                ※ advisor のみ
        ├── docker/        # Docker Compose 定義                   ※ memoir のみ
        └── scripts/       # 実行スクリプト                        ※ memoir のみ
```

## モジュール別ファイルツリー

### easy-agent
```
easy-agent/
├── agents/
│   └── easy-agent.agent.md         # 統合オーケストレーター定義
├── apm.yml                         # 依存: advisor, parliament, taskforce, memoir
├── plugin.json
├── README.md
└── .gitignore
```

### advisor
```
advisor/
├── agents/
│   └── advisor.agent.md            # Advisor エージェント定義 (Opus)
├── skills/
│   └── call-advisor/
│       ├── SKILL.md                # call-advisor スキル
│       └── references/
│           └── prompt-template.md  # 相談プロンプト例集
├── apm.yml                         # 宣言依存なし
├── plugin.json
├── README.md
└── .gitignore
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
│       │   ├── member_message.json      # 議員発言スキーマ
│       │   ├── chairperson_output.json  # 議長出力スキーマ
│       │   └── orchestrator_state.json  # 状態管理スキーマ
│       └── templates/
│           └── stance_definitions.md    # スタンス定義
├── apm.yml                              # 宣言依存なし
├── plugin.json
├── README.md
└── .gitignore
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
├── apm.yml                          # 宣言依存なし
├── plugin.json
├── README.md
└── .gitignore
```

### memoir
```
memoir/
├── skills/
│   └── long-term-memory/
│       ├── SKILL.md                    # スキル定義 (日本語)
│       ├── docker/
│       │   └── docker-compose.yml      # ChromaDB コンテナ
│       └── scripts/
│           ├── _common.py              # 共通ユーティリティ
│           ├── memory_save.py          # 保存
│           ├── memory_search.py        # 検索
│           ├── memory_update.py        # 更新
│           └── memory_delete.py        # 削除
├── apm.yml                             # 宣言依存なし
├── plugin.json
├── README.md
└── .gitignore
```

`memoir` は他パッケージと異なり `agents/` ディレクトリを持たず、`scripts/` 経由で ChromaDB 操作を実行するスキル単独パッケージとして構成される。
