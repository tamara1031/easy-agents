# advisor

エクゼキューター (Sonnet/Haiku) が重要な判断ポイントで高知能なアドバイザー (Opus) に相談し、設計・実装方針の妥当性確認・リスク指摘・エスカレーション判断を受けるパッケージです。単独では動作せず、`call-advisor` スキル経由でサブエージェントとして起動されます。

---

## Advisor パターンとは

**Executor / Advisor 分離**アーキテクチャに基づいています。

| 役割 | モデル | 責務 |
| :--- | :--- | :--- |
| **Executor (エクゼキューター)** | Sonnet / Haiku | ファイル読み取り・検索・コード編集・コマンド実行などの実作業を担う |
| **Advisor (アドバイザー)** | Opus | ツールを呼び出さず、分析と助言のみを行う。計画・修正指示・停止シグナルを返す |

エクゼキューターは実行中に「迷い」を検出すると `call-advisor` スキルを通じて Advisor サブエージェントを起動し、戦略的判断を受け取ります。Advisor の思考プロセスはエクゼキューターのメインコンテキストに蓄積されないため、コンテキストウィンドウを節約しながら高品質な判断を得られます。

---

## 相談トリガー 7 種

エクゼキューターは実行中に常に以下のトリガーを監視します。

| # | トリガー名 | 識別基準 | タイミング |
| :--- | :--- | :--- | :--- |
| 1 | **ゴール未定義** | ゴール未定義・分析困難・外部依存性を新たに検出 | 実行準備前 |
| 2 | **中間状態不明** | 複数の解釈案・アプローチがまだ並行していない（解釈案の選択） | 実行準備前 |
| 3 | **行き詰まり** | 実際に試して失敗した（≥2回の試行後）。どのアプローチを選ぶか、またはこのまま続けて良いか判断できない | 実行中（試行後） |
| 4 | **スコープ肥大化** | 実行中に当初の想定より規模が肥大化し、5 つ以上のアプローチを試して失敗した（3 回以上の試行後） | 実行中 |
| 5 | **不可逆性の懸念** | 実行により元に戻せない、または修正不可能な影響（破壊的・環境変更）を与える可能性がある | 実行前の選択 |
| 6 | **残存リスクの検知** | 実行タスクが成功し、ゴールに達しているかどうかの確信がない | 実行後 |
| 7 | **フェーズ遷移時** | Phase Pipeline のフェーズ間で判断が必要（Phase Gate Advisor トリガー） | 実行後 |

> 定型的な作業・単発の Typo 修正・以前の助言と同じ論点の再試行・Small/軽微タスクでは相談しません。

> **相談回数の上限**: `max_consults = 5`（1タスクあたり）。ガイドラインは 1〜3回。残り回数は `<consult_budget_remaining>` タグで追跡します。

---

## 前提条件・依存関係

`apm.yml` に定義された以下のパッケージが必要です。

| パッケージ | 用途 |
| :--- | :--- |
| [easy-agent](https://github.com/easy-agents/easy-agent) | エージェント基盤 |
| [memoir](https://github.com/easy-agents/memoir) | コンテキスト管理・要約 |
| [parliament](https://github.com/easy-agents/parliament) | 多角的な設計合議（ESCALATE 先） |
| [taskforce](https://github.com/easy-agents/taskforce) | 並列ワークストリーム委譲（ESCALATE 先） |

---

## 使い方

`call-advisor` スキルを通じてエクゼキューターから呼び出します。エクゼキューターが直接 `advisor` エージェントを起動することはできません。

### CLI パターン（`task` ツール利用時）

```bash
task(
  agent_type: "my-copilot:advisor",
  mode: "background",
  name: "advisor-consult-{n}",
  description: "Advisor consultation #{n}",
  prompt: "<XMLプロンプト>"
)
```

完了後は `read_agent` で結果を自動取得します。

### VS Code パターン（`runSubagent` ツール利用時）

```javascript
runSubagent(
  agentName: "advisor",
  description: "Advisor consultation #{n}",
  prompt: "<XMLプロンプト>"
)
```

### XMLプロンプト構造

アドバイザーへのプロンプトは以下の XML タグ構造で構築します。

```xml
<task_summary>
タスクの説明。何を目指し、何に困っているか。
</task_summary>

<current_state>
- どのような状態か？
- 何を試したか？
- どのような制約・環境か？
</current_state>

<actions_taken>
これまでの実行ログ。何を、どう試したか。
</actions_taken>

<consultation_reason>
相談の理由。なぜ今、アドバイザーが必要か。
</consultation_reason>

<!-- フェーズ遷移時のみ追加 -->
<phase_context>
  <current_phase>完了したフェーズ名</current_phase>
  <next_phase>次のフェーズ名</next_phase>
  <task_type>TaskType</task_type>
  <transition_reason>遷移判断に迷う具体的な理由</transition_reason>
  <consult_budget_remaining>残りの相談可能回数</consult_budget_remaining>
</phase_context>

<question>
具体的な質問を 1-3 文で。
</question>

<response_style>concise</response_style>
<!-- いずれか1つを選んで記入: concise（デフォルト）/ detailed / xml -->
<!-- concise: 番号付きステップで100語以内。行動指示優先 -->
<!-- detailed: 分析・根拠・リスクを含む800〜1000トークン -->
<!-- xml: 構造化XMLフォーマット（プログラム処理向け） -->
```

**コンテキスト品質のポイント**

- 関連コードの要約・ディレクトリ構造の抜粋を含める
- 試したアプローチとその結果（成功/失敗のログ）を明記する
- 入力トークンは 500〜1000 トークンを目安にする

---

## 返答フォーマット

Advisor は以下の XML 構造で回答します。

```xml
<analysis>
状況の分析（1-3 文）。エクゼキューターが見落としている観点があれば指摘する。
</analysis>

<recommended_approach>
1. 番号付きステップで具体的かつ実行可能に記述する
2. ファイルパスやメソッド名など即座に行動に移せる具体性を持たせる
</recommended_approach>

<risks>
懸念点や残存リスク。なければ「なし」。
</risks>

<verdict>PROCEED / CORRECT / ESCALATE / STOP</verdict>

<!-- ESCALATE の場合のみ追加 -->
<escalation_target>hierarchy / parliament</escalation_target>
<escalation_reason>
エスカレーションが必要な理由（1-2 文）。
</escalation_reason>
```

### 判定キーワードと executor の行動

| 判定 | 意味 | Executor の次のアクション |
| :--- | :--- | :--- |
| **PROCEED** | 問題なし | `recommended_approach` の内容を参考に現在のフェーズを続行する |
| **CORRECT** | 修正が必要 | `recommended_approach` のステップに従い修正してから続行する |
| **ESCALATE** | エクゼキューター単独では対処困難 | `escalation_target` が示すスキル（`call-taskforce` / `call-parliament`）を起動して委譲する |
| **STOP** | 中断すべき | 実行を停止し、ユーザーに状況を提示して指示を仰ぐ |

### ESCALATE 後のフロー

| `escalation_target` | 委譲先スキル | 用途 |
| :--- | :--- | :--- |
| `hierarchy` | `call-taskforce` | 複数の独立した実装ワークストリームの並列実作業 |
| `parliament` | `call-parliament` | 未解決の設計トレードオフを多角的な合議で解決 |

---

## ファイル構成

```
advisor/
├── apm.yml                          # パッケージ依存関係定義
├── plugin.json                      # パッケージメタデータ
├── agents/
│   └── advisor.agent.md             # Advisor エージェント定義（Opus モデル、tools なし）
└── skills/
    └── call-advisor/
        ├── SKILL.md                 # call-advisor スキル定義（トリガー条件・呼び出し方・ワークフロー）
        └── references/
            └── prompt-template.md   # XMLプロンプトの具体例集（5 パターン）
```
