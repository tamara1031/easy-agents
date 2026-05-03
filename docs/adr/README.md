# Architecture Decision Records (ADR)

easy-agents-hub の主要アーキテクチャ決定の記録。

## 一覧

| ADR | タイトル | ステータス |
|---|---|---|
| [ADR-001](./ADR-001-advisor-pattern.md) | Advisor パターン (Executor/Advisor 分離) | Accepted |
| [ADR-002](./ADR-002-parliament-model.md) | Parliament モデル (多視点合意形成) | Accepted |
| [ADR-003](./ADR-003-hierarchy-execution.md) | 階層型タスク実行 (PIR サイクル) | Accepted |
| [ADR-004](./ADR-004-vector-memory.md) | ベクターDB長期記憶 (ChromaDB) | Accepted |
| [ADR-005](./ADR-005-apm-packaging.md) | APM によるパッケージ管理 | Accepted |
| [ADR-006](./ADR-006-phase-gate-protocol.md) | Phase Gate Protocol | Accepted |
| [ADR-007](./ADR-007-refine-loop-pattern.md) | refine-loop パッケージ分離 (バイアスフリー反復改善) | Accepted |
| [ADR-008](./ADR-008-subagent-return-protocol.md) | サブエージェント返却プロトコルの形式化 (Caller Response Contract) | Accepted |
| [ADR-009](./ADR-009-caller-response-contract-convention.md) | Caller Response Contract Convention (Relay Principle / Two-layer / SSoT) | Accepted |
| [ADR-010](./ADR-010-role-taxonomy.md) | 役割明確化タクソノミー (Skills / Agents / Instructions / Hooks) | Accepted |
| [ADR-011](./ADR-011-hook-specification-format.md) | Hook Specification Format & Trigger Event Vocabulary | Accepted |
| [ADR-012](./ADR-012-physical-role-separation.md) | 役割別物理ファイル分離 (Instructions / Hooks の .apm/ 抽出) | Accepted |
| [ADR-013](./ADR-013-context-budget-protocol.md) | Context Budget Declaration Protocol (call-* スキルのトークン予算宣言) | Accepted |
| [ADR-014](./ADR-014-parallel-orchestration-budget.md) | Parallel Orchestration Budget (連鎖オーケストレーション予算 / cross-skill handoff compression) | Accepted |
| [ADR-015](./ADR-015-dispatch-failure-protocol.md) | Dispatch Failure Protocol (サブエージェント起動失敗の正規処理規約) | Accepted |
| [ADR-016](./ADR-016-memoir-agent-bridge.md) | Memoir Agent Bridge Pattern (VS Code 長期記憶アクセス) | Accepted |
| [ADR-017](./ADR-017-symmetric-output-schema-coverage.md) | Symmetric Output Schema Coverage (全 call-* スキルへの JSON スキーマ適用) | Accepted |
| [ADR-018](./ADR-018-critical-severity-hierarchy.md) | [critical] Severity Tagging for Hierarchy Quality Gates (Hierarchy への二段階収束モデル統一) | Accepted |
| [ADR-019](./ADR-019-parliament-critical-severity.md) | [critical] Severity Tagging for Parliament Quality Gates (Parliament への二段階収束モデル統一) | Accepted |
| [ADR-020](./ADR-020-residual-risks-checklist-bridge.md) | Residual Risks Checklist Bridge (Implement → Verify フェーズ間 non-critical 継承) | Accepted |
| [ADR-021](./ADR-021-canonical-source-section-registry.md) | Canonical Source Section Registry Protocol (アセンブルビュー双方向 CI 保護) | Accepted |
| [ADR-022](./ADR-022-agent-invocation-name-coverage.md) | Agent Invocation Name Coverage (全呼び出しパターンのエージェント名検証) | Accepted |
