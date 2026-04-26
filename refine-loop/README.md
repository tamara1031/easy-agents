# refine-loop パッケージ

成果物（コード・設計・計画）を毎イテレーションで**新規のブランクスレートサブエージェント**にレビューさせ、「実行 → バイアスフリーレビュー → 修正 → 再レビュー」のループを **[critical] 要件が 2 連続で全達成**するまで繰り返して品質を収束させる、反復改善ループ専用パッケージです。

`empirical-prompt-tuning` の実証的改善原理を、プロンプトに限らずあらゆる成果物の品質保証に汎用化したものです。easy-agent の Verify フェーズで自己評価ループ（REVISE）の代わりに呼び出されることを主用途とします。

---

## 設計の核心

> **作成者は自分の成果物を客観視できない。**

そのため refine-loop は以下の不変条件を強制します。

| 不変条件 | なぜ必要か |
| :--- | :--- |
| **毎イテレーションで新規サブエージェントを dispatch** | 前回までの文脈が混入するとバイアスが蓄積する。レビュアーは常にブランクスレートでなければならない |
| **自己評価の禁止** | refine-loop 自身がレビュアーを兼任するとループが無意味化する。`agent` ツールが利用不可な環境では即座に ABORT |
| **[critical] タグはループ開始後に追加・削除しない** | 動かせるゴールは収束判定を破壊する |
| **1 イテレーション = 1 テーマの修正** | 無関係な複数修正を同時に行うと、何が効いたかの因果関係が失われる |

---

## ループ構造

```
[成果物の現在状態]
      ↓
  Iteration N
  ─────────────────────────────────────────
  ① 新規レビュアー dispatch（毎回別エージェント）
     subject + requirements_checklist + task_context を渡す
  ② 構造化レポート収集
     要件達成状況 / Unclear Points / 裁量判断 / Retries
  ③ 収束チェック
     [critical] 未達 0 件 が 2 連続       → CONVERGED
     max_iterations 到達                   → MAX_ITER
     同一 Fix Rule が 3 回以上出現         → ESCALATE
     [critical] タグの追加 / 削除を検知    → ABORT
  ④ [未収束] 最小差分修正を 1 テーマ適用
  ─────────────────────────────────────────
      ↓
  Iteration N+1
```

> **複数条件の同時成立時の優先順位**: `ABORT` > `ESCALATE` > `MAX_ITER` > `CONVERGED`

---

## 入力パラメータ

| パラメータ | 必須 | 型 | 説明 |
| :--- | :--- | :--- | :--- |
| `subject` | ✓ | string | 対象成果物の説明とファイルパス |
| `requirements_checklist` | ✓ | list | 成果物が満たすべき要件リスト。最低 1 つ `[critical]` タグ必須 |
| `task_context` | ✓ | string | 背景・制約・意図（レビュアーが判断に使う文脈） |
| `max_iterations` | — | int | 最大反復数（デフォルト: 3） |

---

## いつ使うか / 使わないか

### 使うべき場面

- easy-agent の **Verify フェーズ**で REVISE が 2 回連続発生し、自己評価で品質が上がらないことが判明した場合
- 設計書・アーキテクチャ決定・重要ロジックなど**外部視点による品質保証が必要**な成果物
- Phase Gate で「なぜ品質が上がらないか」を作成者自身が診断できない場合

### 使わない場面

| 状況 | 代替手段 |
| :--- | :--- |
| 単純な typo 修正・既知パターンの適用 | 自律実行で十分 |
| Explore フェーズの情報収集 | レビューではなく調査が必要 |
| `max_iterations=0` 指定時 | refine-loop を呼ばず直ちに完了 |
| `agent` dispatch ツールがない環境 | refine-loop は Step 0 で ABORT する。easy-agent などトップレベルから直接呼び出す必要がある |

---

## 前提条件・依存関係

`refine-loop` は **APM 宣言依存を持たない独立パッケージ**です（`apm.yml` の `dependencies.apm: []`）。

### 実行時の連携先（宣言依存ではない）

| パッケージ | 連携の種類 | 役割 |
| :--- | :--- | :--- |
| `easy-agent` | 呼び出し元 | Verify フェーズで `call-refine-loop` 経由で起動する |
| 任意の `*.agent.md` | レビュアー候補 | 毎イテレーションで新規サブエージェントとして dispatch される |

---

## 使い方

`agent` ツールで `refine-loop` エージェントを起動します（Skill ツール経由ではなく、サブエージェントとして直接呼ぶ）。

### Claude Code パターン（`agent` ツール）

```
agent(
  subagent_type: "refine-loop",
  description: "Verify: iterative refinement of <subject>",
  prompt: """
    subject: "<対象ファイルパスと成果物の説明>"
    requirements_checklist:
      - "[critical] <必須要件>"
      - "<通常要件>"
    task_context: "<背景・制約・意図>"
    max_iterations: 3
  """
)
```

> **agent ツールが利用不可の場合**: 呼び出し元（easy-agent 等）は REVISE ループ（最大 2 回）にフォールバックし、ユーザーに `[refine-loop 不可: agent ツールなし。自己評価モードで継続します]` と通知する。

---

## 完了レポートのフォーマット

```markdown
## refine-loop 完了レポート

- **ステータス**: CONVERGED / MAX_ITER / ESCALATE / ABORT
- **実行回数**: N / {max_iterations}
- **最終 Accuracy**: XX%（最終イテレーションの値。累積平均ではない。ABORT 時は N/A）

### イテレーション別サマリー
| Iter | SUCCESS/FAILURE | Accuracy | 適用テーマ |
| --- | --- | --- | --- |
| 1 | FAILURE | 60% | <修正テーマ> |
| 2 | SUCCESS | 100% | <修正テーマ> |

### 残存 issues（MAX_ITER / ESCALATE 時）
- <issue>

### Fix Rule レジャー（本ループ内で発見）
- <pattern>

### ABORT / ESCALATE 時の回復ガイダンス
- ABORT（[critical] タグなし）: requirements_checklist に最低 1 つ [critical] タグを付けて再呼び出し
- ABORT（dispatch 不可）: `agent` ツールが利用可能な環境で refine-loop を呼び出す
- ESCALATE: 同一 Fix Rule が 3 回出現 → 設計上の問題。呼び出し元の Phase Gate で Advisory または Parliament へ委譲
```

---

## Fix Rule レジャー

同一の Fix Rule が繰り返し出現する場合、それは成果物の**設計上の問題**を示すシグナルです。3 回出現で `ESCALATE` トリガーとなり、ループ内では解決せず呼び出し元に返します。

### Fix Rule の照合ルール

「同一性」は **表面テキストの一致ではなく根本原因クラスの一致** で判定します。

- **大文字小文字を無視**: `Null Check Missing` と `null check missing` は同一
- **同義表現を同一視**: 「null チェック漏れ」「NPE ガード不足」「null pointer guard」は根本原因クラス `null-safety` として統一
- **判定のヒューリスティクス**: 「同じコード変更で解決できる問題を指している」なら同一クラス
- **パターン名の正規化**: 動詞句ではなく名詞句（例: `null-safety`, `input-validation`, `error-propagation`）を推奨

---

## ファイル構成

```
refine-loop/
├── apm.yml                         # APM パッケージ設定（宣言依存なし）
├── README.md                       # 本ドキュメント
├── .gitignore                      # .claude/ .github/ apm_modules/ を除外
└── .apm/
    ├── agents/
    │   └── refine-loop.agent.md    # refine-loop エージェント定義（user-invocable: false）
    └── skills/
        └── refine-loop/
            └── SKILL.md            # スキル定義（ループ仕様・収束判定・Fix Rule レジャー）
```
