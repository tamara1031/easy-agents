# スキル構成

## スキル一覧

| スキル | モジュール | user-invocable | 呼び出し方 |
|---|---|---|---|
| long-term-memory | memoir | ✅ | ユーザーが「覚えておいて」等と言ったとき |
| call-advisor | advisor | ❌ | エージェントが判断分岐点に達したとき |
| call-parliament | parliament | ❌ | 設計討議・多角的検討が必要なとき |
| call-hierarchy | taskforce | ❌ | 大規模実装タスクを委譲するとき |
| empirical-prompt-tuning | (mizchi/skills) | ✅ | プロンプト改善ループを回すとき |

---

## long-term-memory

**パス**: `memoir/skills/long-term-memory/SKILL.md`

**インフラ**:
- バックエンド: ChromaDB 0.6.3 (Docker)
- コンテナ名: `copilot-memory-chromadb`
- ポート: 18000
- 永続ボリューム: `~/.local/share/copilot-memory/chroma-data`
- 埋め込みモデル: all-MiniLM-L6-v2 (ONNX, クライアントサイド)
- 距離メトリクス: コサイン類似度

**操作**:

| 操作 | スクリプト | 主要引数 |
|---|---|---|
| 保存 | `memory_save.py` | --items / --text / --file, --tags, --dedup |
| 検索 | `memory_search.py` | --query, --n-results, --tags, --json |
| 更新 | `memory_update.py` | --id, --text, --tags, --source |
| 削除 | `memory_delete.py` | --id / --ids / --tag, --confirm |

**テキスト自動分解ルール**:
1. 段落 (二重改行) で分割
2. リスト形式なら各項目に分割
3. 段落 > `max_chunk_size` (デフォルト500文字) ならセンテンス分割

**タグ分類体系**:

| 種別 | 例 |
|---|---|
| domain | backend, frontend, infra |
| task | bugfix, refactor, feature |
| source | user_input, code, docs |
| project | easy-agents-hub, etc. |
| type | concept, fact, procedure |

---

## call-advisor

**パス**: `advisor/skills/call-advisor/SKILL.md`

**コンセプト**: Advisor パターン。実行エージェント (Sonnet/Haiku) が判断分岐点で Opus エージェントに戦略的助言を求める。

**パラメータ**:

| パラメータ | デフォルト | 説明 |
|---|---|---|
| `max_consults` | 5 | タスクあたり最大相談回数 |
| `advisor_mode` | concise | 応答スタイル (concise/detailed) |

**必須プロンプト構造**:

```xml
<task_summary>...</task_summary>
<current_state>...</current_state>
<actions_taken>...</actions_taken>
<consultation_reason>...</consultation_reason>
<phase_context>
  <current_phase>...</current_phase>
  <next_phase>...</next_phase>
  <task_type>...</task_type>
  <transition_reason>...</transition_reason>
  <consult_budget_remaining>...</consult_budget_remaining>
</phase_context>
<question>...</question>
<response_style>concise / detailed / xml</response_style>
```

**相談後のフロー**:
1. Analysis → Execution → Consultation → Integration → Completion

**コンテキスト予算**:
- 入力: 500-1000 tokens
- 出力: 400-700 tokens

---

## call-parliament

**パス**: `parliament/skills/call-parliament/SKILL.md`

**コンセプト**: 議会モデル。大きな目標を複数トピックに分解し、多視点討議でコンセンサスを形成。

**パラメータ**:

| パラメータ | デフォルト | 範囲 | 説明 |
|---|---|---|---|
| `parallelism` | 3 | ≥1 | 同時討議トピック数 |
| `max_rejections` | 3 | ≥1 | トピックあたり最大リジェクト回数 |
| `summary_interval` | 4 | ≥1 | 要約間隔 (発言数) |
| `member_count` | 4 | 4-6 | 議員数 |
| `max_rounds` | 5 | ≥1 | 最大討議ラウンド数 |
| `convergence_threshold` | 2 | ≥1 | 収束判定ラウンド数 |

**ワークフロー (4フェーズ)**:

| フェーズ | 内容 |
|---|---|
| Phase 1 | トピック分解・チェックリスト作成 (ユーザー承認) |
| Phase 2 | 並列討議実行 (background task) |
| Phase 3 | チェックリスト検証 (APPROVED/REJECTED) |
| Phase 4 | 全トピック集約・最終合成 |

**JSON スキーマ**:
- `member_message.json`: 議員発言 (agent_role, stance, target_agent, statement, condition_for_approval)
- `chairperson_output.json`: 議長出力 (action, topic_id, 状態ごとの条件フィールド)
- `orchestrator_state.json`: オーケストレーター状態 (phase, parameters, topics, grand_synthesis)

---

## call-hierarchy

**パス**: `taskforce/skills/call-hierarchy/SKILL.md`

**コンセプト**: 階層型タスク実行。大規模タスクを Plan→Implement→Review サイクルで品質保証しながら実装。

**パラメータ**:

| パラメータ | デフォルト | 説明 |
|---|---|---|
| `parallelism` | 5 | 同時マネージャースロット数 |
| `max_rejections` | 3 | タスクあたり最大リジェクト回数 |
| `member_count` | 3 | 生成メンバー数 (最大5) |
| `checklist_path` | docs/hierarchy-checklist.md | 状態追跡ファイル |

**ワークフロー (4フェーズ)**:

| フェーズ | 内容 |
|---|---|
| Phase 1 | タスクリスト解析・チェックリスト作成・ユーザー承認 |
| Phase 2 | 依存解決 (トポロジカルソート) → ローリング並列実行 |
| Phase 3 | ゲートキーパーレビュー (APPROVE/REVISE) |
| Phase 4 | 最終ゲート実行・成果物集約 |

**タスク分解原則**:
- ✅ 凝集したコンテキスト単位 (「認証モジュール実装」)
- ✅ 独立したエンドポイント単位
- ❌ コード + テストの分割 (コンテキスト重複)
- ❌ ルーティング + ビジネスロジックの分割 (密結合)

**使用しない場合**:
- 1-2ファイル以内の変更
- 15分以内に完了するタスク
- 戦略的設計変更 (Parliament を使う)

**JSON スキーマ**:
- `manager_output.json`: マネージャー出力 (task_id, status, phase, checklist_validation)
- `member_output.json`: メンバー出力 (agent_role, verdict, output, checklist_coverage)
- `orchestrator_state.json`: 状態 (phase, parameters, tasks, grand_summary)

**タスクステータス遷移**:
```
TODO → IN_PROGRESS → IN_REVIEW → APPROVED
                  ↘ REJECTED → TODO (再キュー)
Any → ERROR
```

---

## empirical-prompt-tuning

**パス**: `.claude/skills/empirical-prompt-tuning/SKILL.md`
**ソース**: `mizchi/skills` (外部 GitHub リポジトリ)

**目的**: プロンプト品質の実証的改善。バイアスのないエグゼキューターにシナリオを実行させ、双方向評価でイテレーション。

**ワークフロー**:
0. Iteration 0: 静的整合性チェック (description vs body)
1. ベースライン準備 (シナリオ2-3種 + 要件チェックリスト)
2. サブエージェントへ実行委譲 (新エージェントを毎回dispatch)
3. 実行・自己報告
4. 双方向評価 (executor自己報告 + instruction側計測)
5. 差分適用 (1テーマ/iteration)
6. 再評価
7. 収束チェック (2連続クリアで停止)

**評価軸**:
- Success/Failure (binary)
- Accuracy (要件達成率 %)
- Step count (tool_uses)
- Duration (duration_ms)
- Retry count
- Unclear points (定性)
