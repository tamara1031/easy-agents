# ADR-008: サブエージェント返却プロトコルの形式化

- **ステータス**: Accepted
- **決定日**: 2026-04-26

## コンテキスト

easy-agent はオーケストレーターとして `refine-loop`、`parliament`（Chairperson）、`hierarchy`（Manager）、`advisor` の4種のサブエージェントを呼び出す。各サブエージェントは複数の終了ステータスを返すが、以下の問題が存在した。

1. **Fallback Chain の欠損**: easy-agent の Fallback Chain テーブルに定義されていた失敗ケースは3行のみ（refine-loop ESCALATE、Deliberate 停滞、Plan 破綻）。実際に発生しうる状態（refine-loop の ABORT・MAX_ITER、parliament の MAX_ROUNDS、hierarchy の max_rejections 超過）が未定義のままで、これらが発生した場合の対応方針がなかった。

2. **サイレント障害リスク**: 上記の未定義ケースが発生した場合、easy-agent は明確な対処法を持たず、ループ継続・誤った進行・無限再試行といったサイレント障害に陥るリスクがあった。

3. **一方向のドキュメント**: 各スキル（call-refine-loop 等）は「サブエージェントが何を返すか」を記述していたが、「呼び出し元が何をすべきか」（Caller Response Contract）が定義されていなかった。これにより、スキルの利用者（easy-agent や他のオーケストレーター）は返却ステータスの意味を内部的に推測するしかなかった。

## 決定

**サブエージェント返却プロトコルを形式化し、Fallback Chain と Caller Response Contract を完備する。**

### 1. refine-loop 返却プロトコル

| ステータス | 根本原因 | easy-agent の対応 |
| :--- | :--- | :--- |
| `CONVERGED` | [critical] 要件が2連続で全達成 | 次フェーズへ進む |
| `MAX_ITER` | 反復上限到達・品質未収束（漸進的改善の時間切れ） | ユーザーに残存 issues を提示し、続行/差し戻しをユーザーが選択する。自動進行しない |
| `ESCALATE` | 同一 Fix Rule が3回以上出現（設計上の根本問題） | `Implement` フェーズへ差し戻し。再設計を促す |
| `ABORT` ([critical] なし) | requirements_checklist に [critical] タグが存在しない | checklist を再構築して [critical] タグを付与し refine-loop を再呼び出し。再 ABORT なら STOP |
| `ABORT` (dispatch 不可) | `agent` ツールが利用不可 | REVISE ループ（最大2回）にフォールバック。ユーザーに通知 |

### 2. parliament 返却プロトコル

| ステータス | 根本原因 | easy-agent の対応 |
| :--- | :--- | :--- |
| `AGREED` | 全メンバーが APPROVE または軽微な REVISE | 合意案を採用して `Plan` フェーズへ進む |
| `CONVERGED` | convergence_threshold 達成（議論停滞による収束） | 合意案（残存課題つき）を採用して `Plan` フェーズへ進む |
| `MAX_ROUNDS` | max_rounds に到達（部分合意） | 残存課題を明記した最善合意案を採用し `Plan` へ進む。ユーザーに残存課題と選択肢（続行 / 要件緩和 / Advisory 追加収集）を通知する |
| `max_rejections` 超過 | 検収差し戻しが上限を超過（チェックリスト未達） | parliament が提示した選択肢をユーザーへ転送する。ユーザー選択後に再実行、または STOP |

### 3. hierarchy 返却プロトコル

| ステータス | 根本原因 | easy-agent の対応 |
| :--- | :--- | :--- |
| 全タスク APPROVED | 全チェックリスト達成 | `Verify` フェーズへ進む |
| `max_rejections` 超過 | 検収差し戻しが上限を超過 | hierarchy が提示した選択肢をユーザーへ転送する。ユーザー選択後に再実行、または STOP |

## 設計の根拠

### MAX_ITER vs ESCALATE の区別

この2つは混同されやすいが、根本原因が異なる。

- **MAX_ITER**: 反復によって品質は向上していたが、時間（イテレーション）が尽きた。成果物に構造的欠陥はないが、改善が完了しなかった。ユーザーの裁量で「許容可能な品質」として続行できる可能性がある。
- **ESCALATE**: 同一クラスの問題が3回繰り返された。修正がループしており、成果物に根本的な設計上の問題がある。ユーザー裁量での続行は不適切で、再設計が必要。

### ユーザー転送の設計方針

`max_rejections` 超過や `MAX_ITER` のケースでは、easy-agent がサブエージェントの提示した選択肢をユーザーへ **転送（relay）** する。easy-agent が独自に別の選択肢を提示したり、自動的に次フェーズへ進んだりしない。これにより:

1. ユーザーがオーケストレーション全体の状態を把握できる
2. easy-agent が「失敗した状態を隠蔽」することを防ぐ
3. サブエージェントのフォールバック戦略とオーケストレーターの応答が矛盾しない

### ABORT の2パターン分離

refine-loop の ABORT は根本原因が2種類あり、対応方針が異なる。

| ABORT 種別 | 根本原因 | 対応 |
| :--- | :--- | :--- |
| `[critical]` タグなし | 呼び出し側のチェックリスト構築ミス | checklist 修正後に再呼び出し（再試行可能） |
| dispatch 不可 | 実行環境に `agent` ツールがない | REVISE フォールバック（再試行不可） |

これらを同一 ABORT として扱うと、dispatch 不可の環境で checklist 修正ループに陥るリスクがある。

## 変更ファイル

| ファイル | 変更内容 |
| :--- | :--- |
| `easy-agent/.apm/agents/easy-agent.agent.md` | Fallback Chain テーブルに refine-loop ABORT/MAX_ITER、parliament MAX_ROUNDS、hierarchy max_rejections 超過を追加。Verification Criteria の Verify 行を返却ステータス別対応に更新。Transition Graph を Fallback Chain への参照に簡潔化 |
| `refine-loop/.apm/skills/call-refine-loop/SKILL.md` | 「呼び出し元の応答コントラクト」セクションを追加。全返却ステータスのアクションと MAX_ITER/ESCALATE の違いを明文化 |

## 結果として得られる特性

- **完全性**: easy-agent は全サブエージェントのすべての返却ステータスに対して明示的な対応方針を持つ
- **透明性**: 失敗・部分成功のケースでユーザーへの報告が義務付けられ、サイレント障害を排除
- **対称性**: スキルのドキュメントが「サブエージェントの出力」と「呼び出し元の応答」の両方を定義する双方向のコントラクトになる

## 今後の課題

- `call-parliament/SKILL.md` と `call-hierarchy/SKILL.md` にも同様の Caller Response Contract を追加することで対称性をさらに高められる（本 ADR のスコープ外）
- `advisor` の返却ステータス（PROCEED / CORRECT / ESCALATE / STOP）については、easy-agent の既存セクション「Advisory 判定後の処理フロー」が既に対応しており、今回の変更対象から除外した
