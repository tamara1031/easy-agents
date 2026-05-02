# 定期バッチ実行ガイド: Epic 単位の安全な価値提供

このドキュメントは、定期バッチで作業ブランチを更新しながら、1つの本質的テーマ（Epic）を安全に完了するための実務ガイドです。

## 1. Pre-flight Sync

1. `git fetch origin main`
2. `git merge origin/main`
3. 競合発生時はマーカーを解消し、解決コミットを作成
4. 依存マニフェスト変更時は依存を再インストール
5. テスト/リンターを実行し、意味的競合を検知

> `origin` が未設定の環境では、`git remote add origin <url>` を先に行う。

## 2. Epic 設計（2〜4ステップ）

- テーマは **1つだけ**選ぶ（Cohesion）
- 各ステップは独立コミット可能に分割
- 各ステップで要件チェックリストを定義
  - 最低1つを `[critical]` にする

### チェックリスト雛形

```md
- [ ] [critical] システム要件に直結する条件
- [ ] 仕様整合
- [ ] テスト追加/更新
- [ ] ドキュメント更新
```

## 3. Empirical Improvement Loop

各ステップで次を繰り返す:

1. **Execution**: 実装/文書化
2. **Quantitative Evaluation**: テスト・リンター・フォーマッター
3. **Trace & Reflection**:
   - Understanding / Planning / Execution / Formatting の4フェーズで詰まりを同定
   - `Issue / Cause / General Fix Rule` を記録
   - 暗黙補完（裁量埋め）を明示
4. **General Fix Rule 適用**: 対症療法でなく、再発防止ルールとして修正
5. **Convergence 判定**: 新規課題が止まり、[critical] 達成ならコミット

## 4. Just-in-Time Sync

Push直前に再同期する:

1. `git fetch origin main`
2. `git merge origin/main`
3. 競合解消後に **全テスト再実行**

## 5. Failure Circuit Breakers

- 3イテレーションで収束しない場合は該当ステップを撤退
- マージ意図が読めない競合は `git merge --abort` し、人間へエスカレーション
- 成功コミットのみを残す

## 6. Handoff レポート項目

- 🎯 テーマと変更意図
- 👣 要件達成度とコミットログ
- 📖 Failure pattern ledger（General Fix Rule 一覧）
- ⚠️ レビュワー重点確認箇所（暗黙補完含む）
