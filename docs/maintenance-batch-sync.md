# 定期バッチ運用 Runbook

このドキュメントは、定期バッチで作業ブランチを更新し、変更を安全に出荷するための最小手順を定義する。

## 目的

- main 追従漏れによる差分衝突を減らす。
- 変更の妥当性を「主観」ではなくチェックリストと実行ログで確認する。
- 失敗時の撤退基準を明文化し、壊れた状態の push を防ぐ。

## 実行フロー

### 1) Pre-flight Sync

1. `git fetch origin main`
2. `git merge origin/main`
3. 競合発生時はマーカーを解消して `git add` + `git commit`
4. 依存定義ファイルに差分があれば依存を再インストール
5. テスト/リンターを実行し、意味的競合を確認

> 補足: `origin` や `main` が存在しないローカル検証環境では、このフェーズを「未実施」と明示して記録する。

### 2) Theme Planning

1. 現在差分を確認し、今回扱うテーマを1つに固定する。
2. テーマを 2〜4 ステップに分割する。
3. 各ステップに要件チェックリストを定義し、最低1件の `[critical]` を含める。

### 3) Empirical Loop

各ステップで以下を繰り返す:

- 実装
- 定量評価（テスト/リンター/フォーマッター）
- 定性評価（Understanding / Planning / Execution / Formatting の4観点）
- `Issue / Cause / General Fix Rule` の抽出
- Fix Rule を反映した再実装

収束条件:

- 新しい Fix Rule が出なくなる
- `[critical]` を満たし、テストが成功

収束したステップは Conventional Commits で個別コミットする。

### 4) Just-in-Time Sync

1. `git fetch origin main`
2. `git merge origin/main`
3. 競合があれば解消後に全テスト再実行

### 5) Push & Handoff

1. `git push -u origin HEAD`
2. レビュー向けに以下を報告する:
   - テーマと変更意図
   - 要件達成度とコミット順序
   - Failure pattern ledger
   - レビュワーへの重点確認依頼

## Circuit Breakers

- テーマ外の「ついで修正」を禁止する。
- 同一ステップで 3 イテレーション以上収束しない場合はロールバックする。
- マージ意図が不明な競合は `git merge --abort` し、人間に判断を依頼する。

## 最低限の実行ログテンプレート

```md
## Batch Run Log (YYYY-MM-DD)
- Theme:
- Step list:
  - Step 1:
    - [critical]
    - [ ] check 1
  - Step 2:
    - [critical]

### Validation
- command:
- result:

### Failure pattern ledger
- Issue:
- Cause:
- General Fix Rule:

### Reviewer focus
- discretionary fill-ins:
```
