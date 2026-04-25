# モジュール間依存関係

## 依存グラフ

```
                     ユーザー
                        ↓
              ┌──────────────────┐
              │   easy-agent     │  (統合オーケストレーター・user-invocable)
              └────────┬─────────┘
                       │ APM 宣言依存
        ┌──────────────┼──────────────┬──────────────┐
        ↓              ↓              ↓              ↓
  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ advisor  │  │parliament│  │taskforce │  │  memoir  │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

`easy-agent` が他 4 モジュールを APM 宣言依存として束ねる。サブモジュール同士は APM レベルでは独立しており、実行時の呼び出し関係（後述）でのみ結びつく。

## APM 宣言依存関係

各モジュールの `apm.yml` の `dependencies.apm` フィールドに基づく宣言依存。

| モジュール | 宣言依存 (apm) |
|---|---|
| `easy-agent` | `advisor`, `parliament`, `taskforce`, `memoir` |
| `advisor` | なし |
| `parliament` | なし |
| `taskforce` | なし |
| `memoir` | なし |

ルートワークスペース (`apm.yml`) は `easy-agent` のみを宣言依存とし、`easy-agent` の依存解決を通じて全モジュールがインストールされる。

## エージェント間の呼び出し関係

宣言依存とは別に、実行時にはエージェントが互いをサブエージェントとして起動する。

### easy-agent → サブエージェント

| 呼び出し先 | トリガー条件 | スキル |
|---|---|---|
| `parliament-chairperson` | Deliberate フェーズ / 設計判断 | call-parliament |
| `hierarchy-manager` | Implement フェーズ (4ファイル以上) | call-hierarchy |
| `advisor` | Phase Gate / 判断が必要な局面 | call-advisor |

### advisor → エスカレーション先

`advisor` 自体はツールを持たないが、`<verdict>ESCALATE</verdict>` を返した際に呼び出し元エージェントへ次の委譲先を指示する。

| 判定 | エスカレーション先 |
|---|---|
| ESCALATE (並列実装) | hierarchy (taskforce) |
| ESCALATE (設計合意) | parliament |

### parliament の内部構造

```
call-parliament (Orchestrator)
  └─→ parliament-chairperson
        └─→ parliament-member × N (4〜6名)
              └─→ advisor (任意 / 1回/名限定)
```

### taskforce の内部構造

```
call-hierarchy (Orchestrator)
  └─→ hierarchy-manager
        └─→ hierarchy-member (Planner)
        └─→ hierarchy-member (Implementer)
        └─→ hierarchy-member (Reviewer)
        └─→ advisor (任意)
```

## ツール依存関係

| エージェント | 使用ツール |
|---|---|
| easy-agent | read, edit, search, execute, agent, todo |
| advisor | read, search (実行系ツール不可) |
| parliament-chairperson | read, search, agent |
| parliament-member | read, edit, search, execute, agent |
| hierarchy-manager | read, search, agent, todo |
| hierarchy-member | read, edit, search, execute, agent |

## 外部インフラ依存

| モジュール | 外部依存 |
|---|---|
| memoir | Docker, ChromaDB 0.6.3, chromadb Python パッケージ, ONNX runtime |
| その他全モジュール | なし (純粋な Markdown / JSON 定義) |
