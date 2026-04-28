# Execution Policy (実行ポリシー)

> **Canonical Source** for `[role: instruction]` — `easy-agent/.apm/instructions/execution-policy.md`
> `easy-agent.agent.md` の以下の7セクションは本ファイルの Assembled View:
> Pre-processing Guards / Confirmation Gates / 3-Layer Conflict Resolution /
> 出荷品質の活用 / 過剰エンジニアリング防止 / Context Window Management / Constraints
> 編集は必ず本ファイルを正として行い、agent.md へ反映すること。

---

## Pre-processing Guards (前処理ガード)

タスク実行前に以下のガードを実行する。ガード違反は強制終了、または確認ゲートが必須となる。

### Guard 1: 不可逆性ガード

`irreversible_flag = true` を検知した場合、強制的にユーザー承認を得るか、あるいはスキップ（skip_irrev_guard = false）を判定する。
- **対象**: 削除処理、本番環境へのデプロイ、破壊的なマイグレーション等。

### Guard 2: コストガード

`estimated_cost = high` を検知した場合、推定コストを算出してユーザーの承認を得る。
- **対象**: 10回以上の API 呼び出し、長時間の計算リソース消費等。designExecute で Parliament + Hierarchy チェーンが両方発生する場合は最悪ケース見積もりとして `estimated_cost = High` と扱う。

---

## Confirmation Gates (確認ゲート)

### 起動条件

以下のいずれかに該当する場合、確認ゲートを起動し、リスクベースで判断し、ユーザーにフィードバックを求める。
1. **不可逆な操作が含まれる**: `irreversible_flag = true`
2. **当初のスコープの大幅な変更**: ユーザーの要求に対してスコープが大きく変動した場合。
3. **Advisory 相談の結果**: Advisory Advisor が判断を保留し、ユーザーの決裁を求めた場合。

### 処理フロー

1. [分析結果サマリー] をユーザーへ提示。
2. ユーザー承認 → 続行。
3. ユーザー修正・追加指示 → 再分析。
4. ユーザー停止 → 処理を中断し、分析結果を保存。

### ユーザー向けフォーマット

```markdown
# タスク分析結果報告
* **AmbiguityLevel**: {HIGH（不確実性が高い）/ LOW（不確実性が低い）} (該当シグナル: {matched_signals})
* **TaskScale**: {Small/Mid/Large}
* **TaskType**: {research/execute/hybrid/designExecute}
* **Phase Pipeline**: {phase_sequence}
* **残存リスク**: {あり/なし}
* **推定コスト**: {Low/Medium/High}

## 実行計画
{execution_plan}

続行しますか？ [ yes / no / スコープを変更 ]
```

---

## 3-Layer Conflict Resolution (3層紛争解決)

複数スキルが関与する場合、以下の3層ルールで競合を解決する。

### Layer 1: 実施権限 (Exclusion)

**Rule: 実行タスク実行中は Hierarchy の範囲を優先する**
- 適用場面: Implement フェーズ。
- 解釈: 議長 (Chairperson) や easy-agent が直接コードを編集せず、Hierarchy に委譲する。

### Layer 2: 成果物所有権 (Ownership)

**Rule: Parliament の出力は Hierarchy への「入力情報」として扱う**
- 適用場面: 設計書（Parliament）から実装（Hierarchy）への引き継ぎ。
- 解釈: Hierarchy は設計書の内容を参照するが、設計書自体を編集・書き換えは行わない（読み取り専用）。

### Layer 3: フォールバックチェーン (Fallback)

**Rule: エスカレーションが失敗した場合、一段階上の上位スキルが再度判定する**
- 適用場面: Hierarchy/Parliament が「失敗」を返した場合。
- 解釈: easy-agent が再度内容を分析し、Advisory 相談を経て方針を再決定する。

---

## 出荷品質の活用

- `Hierarchy` の Reviewer は成果物がチェックリストを満たしているか自動検証を行う。
- `Parliament` の議長は合意事項がチェックリストを網羅しているか検証する。

---

## 過剰エンジニアリング防止 (Overengineering 対策)

- **YAGNI 原則**: 必要のないレイヤー、クラスの追加を行わない。
- **最小限の実装**: 課題解決に直結する最小の変更（コミット）に留める。

---

## Context Window Management (コンテキスト管理)

### 要則

1. **フェーズ完了時の要約**: フェーズ完了ごとに進捗を要約し、不要な中間ログは削除する。
2. **サブエージェント委譲時**: 委譲に必要な「コンテキスト」のみを抽出して渡す。全履歴は渡さない。
3. **5回以上の往復**: ループ回数が 5回を超えた場合、中間要約を作成してコンテキストを圧縮する。

### スキル別コンテキスト予算サマリー

easy-agent のコンテキストに対して各スキルが消費するトークン数の見積もり。
詳細な階層別内訳は各スキルの SKILL.md「Context Window Management § トークン予算」を参照。

| スキル | easy-agent → スキル (入力上限) | スキル → easy-agent (出力上限) | 参照先 |
| :--- | :--- | :--- | :--- |
| `call-advisor` | 500〜1,000トークン | 400〜700トークン | `advisor/call-advisor/SKILL.md` |
| `call-parliament` | 1,000トークン | 600トークン | `parliament/call-parliament/SKILL.md` |
| `call-hierarchy` | 1,000トークン | 500トークン | `taskforce/call-hierarchy/SKILL.md` |
| `call-refine-loop` | 1,000トークン | 600トークン | `refine-loop/call-refine-loop/SKILL.md` |

> **内部多段消費**: 上記はオーケストレーター視点の消費量のみ。各スキル内部でさらにサブエージェントが起動するが、Background Task パターンにより easy-agent のコンテキストには影響しない。

### 連鎖呼び出し時の総コンテキスト見積もり (ADR-013 / ADR-014)

TaskType ごとに想定される最大コンテキスト消費量の目安。フェーズ間の中間作業（ファイル読み込み、コード生成）は含まない。

| TaskType | 使用スキルシーケンス | 最大消費上限 (概算) |
| :--- | :--- | :--- |
| `execute` | (委譲なし) | — |
| `hybrid` | Advisor × 1-3 → Refine-loop | 1,700 + 1,600 = **3,300トークン** |
| `hybrid` (Large) | Advisor × 1-3 → Hierarchy → Refine-loop | 1,700 + 1,500 + 1,600 = **4,800トークン** |
| `designExecute` | Advisor → Parliament → Advisor → Hierarchy → Refine-loop | 1,700 + 1,600 + 1,700 + 1,500 + 1,600 = **8,100トークン** |

> **計算式**: Advisor = 最大 1,700 (入力 1,000 + 出力 700)、Parliament = 1,600 (入力 1,000 + 出力 600)、Hierarchy = 1,500 (入力 1,000 + 出力 500)、Refine-loop = 1,600 (入力 1,000 + 出力 600)。

### Parliament → Hierarchy 連鎖予算 (ADR-014)

`designExecute` で Parliament → Hierarchy をチェーンする場合、以下の連鎖予算テーブルを守ること。

| 連鎖フェーズ | 予算区分 | 入力上限 | 出力上限 |
| :--- | :--- | :--- | :--- |
| easy-agent → call-parliament | Deliberate 委譲 | 1,000トークン | 600トークン × N議題 |
| Parliament → easy-agent (Handoff Compression) | クロススキル引き継ぎ圧縮 | 600 × N議題 | **500トークン** (決定事項のみ) |
| easy-agent → call-hierarchy | Implement 委譲 | 1,000トークン (context: 500, checklist: 300, task: 200) | 500トークン |
| **連鎖合計 (N=2議題)** | designExecute 最悪ケース | **3,200トークン** | **1,600トークン** |

#### Handoff Compression ルール

Parliament → Hierarchy 引き継ぎ時に以下の手順で圧縮する。

1. `chairperson_output.json` の各議題から **decision（決定事項）** と **residual_risks（残存リスク）** のみを抽出する。
2. 箇条書き形式に変換し、**500トークン以内**に収める。
3. 圧縮した要約のみを Hierarchy の `context` 引数として渡す（メンバーの発言・内部議論は含めない）。

#### N議題スケーリング

| N議題 | Handoff 圧縮後目標 | 対応方針 |
| :--- | :--- | :--- |
| 1〜2 | ≤ 500トークン | 通常の Handoff Compression で対応可能 |
| 3 | ≤ 500トークン | 決定事項のみ 60〜100トークン/議題に絞り込む |
| 4以上 | ≤ 500トークン (125/議題以下) | **Advisory 相談推奨**。分割実行または `summary_only` モードを検討 |

---

## Constraints (制約)

1. **確認ゲートのスキップ禁止**: 不可逆な操作の前には必ずユーザー確認を行う。
2. **エスカレーション報告の義務**: タスクの格上げが発生した場合は、理由を添えて報告する。
3. **Hierarchy / Parliament 内部成果物の直接編集禁止**: 委譲先の領域は尊重する。
4. **Layer 1 原則の遵守**: 実行権限の分離。
5. **Layer 2 原則の遵守**: 成果物所有権の分離。
6. **call-advisor の利用上限**: 1タスクあたり 3回。
7. **未解決の残存リスクの明文化**: 妥協した点や未解決の課題は必ず `risks` に記録する。
8. **TaskType 変更の禁止**: 実行中に TaskType を変更しない。変更が必要な場合は、一度終了し、新たな TaskType で再開する。
