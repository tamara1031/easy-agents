# ADR-002: Parliament モデル (多視点合意形成)

- **ステータス**: Accepted
- **決定日**: 2026-04-25

## コンテキスト

アーキテクチャ設計・技術的方針決定では、単一視点の判断では見落としが生じやすい。特に以下の懸念を並行して考慮する必要がある:
- 機能推進 (Advocate)
- 品質・リスク (Reviewer)
- 法規制・倫理・セキュリティ (Compliance)
- コスト・工期 (Pragmatist)

## 決定

**4ペルソナ固定 + 任意追加の議会モデル**を採用する。

- 必須: Advocate, Reviewer/Critic, Compliance, Pragmatist の4名
- 任意: Domain Expert, User Advocate, Performance Engineer, Security Specialist
- 議長 (Chairperson) がトピック分解・議事進行・コンセンサス判定を担当
- 各議員はスタンス (PROPOSE/CRITIQUE/APPROVE/REVISE) で構造化発言
- CRITIQUE は必ず `condition_for_approval` を付与 (建設的批判の強制)

## 結果

### メリット
- 設計の死角を組織的に排除
- CRITIQUE に承認条件を必須化することで建設的議論が強制される
- エビデンス階層 (Tier1: ソースコード → Tier4: 外部ドキュメント) による根拠の透明化
- コンテキストウィンドウ管理 (≤500トークン/議員/ラウンド) で長時間討議に対応

### デメリット・リスク
- シンプルな決定に対して議会は過剰なコスト
- メンバーが5回でも収束しない場合の MAX_ROUNDS 強制終了時、未解決事項の扱い
- 議長の「収束判定」が甘いと議論がループする

## 代替案

- **単純多数決**: 速いが少数意見が消える
- **専門家1名に委任**: 効率的だが視点の偏りが生じる

## 利用推奨

- call-hierarchy の代わりに call-parliament を使うべき場面: 「何を作るか」「方針は何か」の設計決定
- call-parliament の代わりに call-hierarchy を使うべき場面: 「どう実装するか」が既知の実装タスク
