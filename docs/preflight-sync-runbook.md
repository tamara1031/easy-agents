# Pre-flight Sync Runbook

定期バッチ実行で作業ブランチを `main` に追従させるための最小手順。

## 目的

- 取り込み漏れを防ぎ、実装着手前に差分の土台を固定する。
- マージ後の意味的衝突（テスト失敗）を早期に検知する。

## Step 1: リモートと `main` の存在確認 **[critical]**

```bash
git remote -v
git branch --all
```

- `origin` が未設定なら、以降の `fetch` は失敗する。
- `main` が存在しない場合は、運用上の正規ブランチ名（例: `master`, `trunk`）を確認して手順を読み替える。

## Step 2: 最新取り込み

```bash
git fetch origin main
git merge origin/main
```

### 競合時

- 意図が読み取れる場合のみ解消してコミット。
- 意図不明なら `git merge --abort` で中断し、人間レビュワーへエスカレーションする。

## Step 3: 依存関係の同期

マニフェスト差分を確認し、変更がある場合のみインストールを実行する。

```bash
git diff --name-only HEAD~1..HEAD
```

例:

- Node.js: `npm install` / `pnpm install`
- Go: `go mod tidy`
- Python: `pip install -r requirements.txt`

## Step 4: 意味的競合の検知 **[critical]**

プロジェクトで定義済みのテスト・リンターを実行し、マージは成功していてもロジックが壊れていないか確認する。

```bash
# 例
npm test
npm run lint
```

## Step 5: Push 直前の再同期

長時間作業後は Push 直前に再度実行:

```bash
git fetch origin main
git merge origin/main
```

競合解消後は、**必ず**テストとリンターを再実行する。

## 失敗パターン台帳テンプレート

- Issue:
- Cause:
- General Fix Rule:

同一パターンが反復した場合は、対症療法ではなく設計ルール化を優先する。
