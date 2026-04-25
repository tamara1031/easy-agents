# モジュール間依存関係

## 依存グラフ

```
                     ユーザー
                        ↓
              ┌─────────────────┐
              │   easy-agent    │  (統合オーケストレーター)
              │  user-invocable │
              └────────┬────────┘
          ┌────────────┼───────────┐
          ↓            ↓           ↓
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │  advisor │  │parliament│  │ taskforce│
   │  (Opus)  │  │          │  │          │
   └────┬─────┘  └──────────┘  └──────────┘
        │
        └──→ ESCALATE → parliament / taskforce

memoir は独立モジュール (advisor経由で参照のみ)
```

## APM 宣言依存関係

| モジュール | apm.yml 依存宣言 |
|---|---|
| easy-agent | empirical-prompt-tuning (mizchi/skills) |
| advisor | easy-agent, memoir, parliament, taskforce |
| parliament | empirical-prompt-tuning (mizchi/skills) |
| taskforce | empirical-prompt-tuning (mizchi/skills) |
| memoir | empirical-prompt-tuning (mizchi/skills) |

> **注意**: advisor の apm.yml は easy-agent/memoir/parliament/taskforce に固定コミットハッシュで依存を宣言している。他モジュールは互いに宣言依存を持たない。

## エージェント間の呼び出し関係

### easy-agent → サブエージェント

| 呼び出し先 | トリガー条件 | スキル |
|---|---|---|
| `parliament-chairperson` | Deliberate フェーズ / 設計判断 | call-parliament |
| `hierarchy-manager` | Implement フェーズ (4ファイル以上) | call-hierarchy |
| `advisor` | Phase Gate / 判断が必要な局面 | call-advisor |

### advisor → エスカレーション先

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
| memoir | Docker, ChromaDB 0.6.3, chromadb Python package, ONNX runtime |
| その他全モジュール | なし (純粋なMarkdown/JSON定義) |
