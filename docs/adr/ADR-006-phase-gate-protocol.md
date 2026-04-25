# ADR-006: Phase Gate Protocol

- **ステータス**: Accepted
- **決定日**: 2026-04-25

## コンテキスト

エージェントが複数フェーズを自律的に実行すると、フェーズ間の品質チェックが省略されがちで、後続フェーズに欠陥が伝播するリスクがある。また、フェーズ遷移のタイミングでユーザー確認や外部専門家への相談が必要な場合がある。

## 決定

**Phase Gate Protocol** を easy-agent の中心的な実行制御機構として採用する。

### フェーズ定義

```
Explore → Deliberate → Plan → Implement → Verify → Synthesize
```

### ゲートステータス

| ステータス | 処理 |
|---|---|
| APPROVED | 次フェーズへ進む |
| REVISE | 同フェーズの修正・再実行 |
| DELEGATE | 専門エージェントへ委譲 (Parliament/Hierarchy) |
| LOOPBACK | 前フェーズへ戻る |
| ESCALATE | ユーザー確認が必要 |

### ゲート通過条件の例

- Explore → Plan: 目標・成果物・変更範囲が確定していること
- Implement → Verify: 全チェックリスト項目が COVERED であること
- Verify → Synthesize: テスト・型チェックがパスしていること

## 結果

### メリット
- フェーズ間で欠陥が伝播しない
- Advisor 相談のタイミングを Phase Gate に集約 (散発的相談を防ぐ)
- ESCALATE により高リスク操作で必ずユーザーが介在する

### デメリット・リスク
- ゲートが厳しすぎると小タスクでもオーバーヘッドが大きい
- REVISE ループが max_iterations に達したときの ESCALATE 判断基準が曖昧になりやすい

## 関連決定

- ADR-001: Phase Gate で advisor を相談するタイミングを定義
- ADR-003: hierarchy は内部に独自の PIR サイクルを持つため、Phase Gate とは別レイヤー
