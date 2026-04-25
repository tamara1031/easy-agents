# ADR-003: 階層型タスク実行 (PIR サイクル)

- **ステータス**: Accepted
- **決定日**: 2026-04-25

## コンテキスト

大規模な実装タスク (4ファイル以上, 15分超) を単一エージェントで処理すると:
- コンテキストウィンドウ枯渇
- 計画と実装の責任が混在して品質が下がる
- 部分的な失敗時の再試行コストが大きい

## 決定

**Generator-Verifier パターンによる Plan→Implement→Review (PIR) サイクル**を採用する。

- Manager が全体を統括 (実装は行わない)
- Planner/Implementer/Reviewer を分離した専門メンバーとして動的生成
- Implementer → Reviewer の内部フィードバックループ (最大5回)
- 各メンバーは独立したコンテキストウィンドウで実行 (コンテキスト汚染防止)
- JSON スキーマによる明確な出力契約 (member_output.json, manager_output.json)

## 結果

### メリット
- Reviewer による客観的な品質ゲート (作成者がレビューしない)
- コンテキスト分離で長大タスクに対応
- チェックリスト駆動で "YAGNI 違反" "テストハック" を構造的に検出
- 並列実行 (parallelism=5) でスループット向上
- タスクステータス追跡 (TODO/IN_PROGRESS/IN_REVIEW/APPROVED/REJECTED/ERROR) で再開対応

### デメリット・リスク
- オーバーヘッド大: 1-2ファイル変更でも全サイクルが走る
- max_rejections 到達後のフォールバック設計が必要
- Reviewer が過剰厳格だとループが止まらない

## 代替案

- **直接実装**: 単純なタスクは階層不要 (easy-agent が Small/Mid は直接処理)
- **並列実装のみ**: Review なしで速いが品質保証なし

## 使い分け基準

| 規模 | TaskScale | 推奨 |
|---|---|---|
| 1ファイル | Small | easy-agent が直接実装 |
| 1-3ファイル | Mid | easy-agent が直接実装 |
| 4ファイル以上 | Large | call-hierarchy に委譲 |
