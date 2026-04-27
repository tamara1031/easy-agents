# ADR-011: 役割別物理ファイル分離 (Instructions / Hooks の .apm/ 抽出)

- **ステータス**: Accepted
- **決定日**: 2026-04-27

## コンテキスト

ADR-010 で easy-agents-hub の設計成果物を4役割（Skills / Agents / Instructions / Hooks）に分類するタクソノミーを確立し、`easy-agent.agent.md` の各 H2 セクションに `[role: ...]` アノテーションを付与した。その際 ADR-010 は「物理的な抽出は本タクソノミーが定着した後の独立 ADR で扱う」と明示的に先送りした。

現状の問題:

1. **単一ファイルへの責務混在**: `easy-agent.agent.md` (578 行) は `[role: agent identity]` / `[role: agent capability]` / `[role: instruction]` / `[role: hook]` の4役割を1ファイルに同居させている。役割をアノテーションで可視化しただけでは、編集者が「どのセクションを変えると何に影響するか」を直感的に把握できない。

2. **再利用不可能なルール**: `[role: instruction]` タグの付いた7セクション（Pre-processing Guards, Confirmation Gates, 3-Layer Conflict Resolution, 出荷品質の活用, 過剰エンジニアリング防止, Context Window Management, Constraints）は、本来 easy-agent 以外のエージェントにも適用すべき原則だが、easy-agent.agent.md に埋め込まれているため参照・再利用の手段がない。

3. **Hooks のアイデンティティ不在**: `Auto-Memory Protocol` は発火条件（SessionStart / PhaseGateComplete / 15-exchange 経過）と動作（memoir recall/save）が明確に定義された Hook だが、Agent 定義ファイルの末尾に同居しているため「いつ・何が発火するか」の見通しが悪い。

4. **APM の構造的空白**: `.apm/` ディレクトリは `agents/` と `skills/` しか定義されておらず、Instructions と Hooks のための物理レイアウトが存在しない。新規エージェント追加時に「ルールをどこに書くか」の基準がない。

5. **ADR-009 残 TODO**: easy-agent.agent.md の Fallback Chain テーブルと各 `call-X/SKILL.md` の Caller Response Contract が独立して記述されており、ドリフト検出の仕組みが未整備。

## 決定

### 1. `.apm/` に `instructions/` と `hooks/` ディレクトリを追加する

```
<package>/.apm/
├── agents/            # (既存) エージェント定義 (*.agent.md)
├── skills/            # (既存) スキル定義 (SKILL.md + 付属ファイル)
├── instructions/      # [NEW] 普遍ルール・制約 (*.md)
└── hooks/             # [NEW] イベント駆動定型処理 (*.md)
```

- `instructions/` 配下の各ファイルは「常時有効な規範・原則・制約」を記述する。
- `hooks/` 配下の各ファイルは「発火条件 + 動作」のペアを記述し、ファイル名はイベント名を基準とする（例: `session-start.md`, `phase-gate.md`）。
- 両ディレクトリは APM がビルド対象として認識するまでの間、**git 追跡対象のソースファイル**として扱う。

### 2. `easy-agent.agent.md` の役割は「assembled view」とする

APM が `.apm/instructions/` や `.apm/hooks/` からのインライン展開をサポートするまでの移行期間中、以下の方針を採る:

- `easy-agent/.apm/instructions/execution-policy.md` と `easy-agent/.apm/hooks/session-start.md` を **Canonical Source（正）** とする。
- `easy-agent.agent.md` の該当セクションは **Assembled View（参照コピー）** として残し、Role Taxonomy ブロックに「このファイルは assembled view である」と明記する。
- 編集者は必ず Canonical Source を編集し、agent.md の対応セクションに反映させる義務を負う。
- CI が Canonical Source ファイルの存在を検証する（内容ドリフトの検出は将来課題）。

### 3. 抽出対象セクションの分類

| セクション | 役割 | 抽出先 |
| :--- | :--- | :--- |
| Pre-processing Guards | `[role: instruction]` | `easy-agent/.apm/instructions/execution-policy.md` |
| Confirmation Gates | `[role: instruction]` | `easy-agent/.apm/instructions/execution-policy.md` |
| 3-Layer Conflict Resolution | `[role: instruction]` | `easy-agent/.apm/instructions/execution-policy.md` |
| 出荷品質の活用 | `[role: instruction]` | `easy-agent/.apm/instructions/execution-policy.md` |
| 過剰エンジニアリング防止 | `[role: instruction]` | `easy-agent/.apm/instructions/execution-policy.md` |
| Context Window Management | `[role: instruction]` | `easy-agent/.apm/instructions/execution-policy.md` |
| Constraints | `[role: instruction]` | `easy-agent/.apm/instructions/execution-policy.md` |
| Auto-Memory Protocol | `[role: hook]` | `easy-agent/.apm/hooks/session-start.md` |

### 4. CI による構造検証（ADR-009 残 TODO への対処を含む）

以下の Lint ステップを `.github/workflows/check.yml` に追加する:

#### Lint A: instructions/hooks canonical source の存在検証

`*.agent.md` ファイル中に `[role: instruction]` タグを持つ H2 セクションが1つ以上存在するパッケージには、`<package>/.apm/instructions/` ディレクトリに少なくとも1つの `.md` ファイルが存在しなければならない。同様に `[role: hook]` タグを持つ場合は `<package>/.apm/hooks/` に少なくとも1つの `.md` ファイルが必要。

#### Lint B: Fallback Chain ↔ Caller Response Contract ステータス整合性 (ADR-009 残 TODO)

`easy-agent.agent.md` の Fallback Chain テーブルに記載されたサブエージェントステータス (`refine-loop`, `parliament`, `hierarchy`) が、対応する `call-X/SKILL.md` の Caller Response Contract テーブルに存在することを検証する。easy-agent が参照しているステータス語がスキル側に定義されていない場合はビルドエラーとする。

## 設計の根拠

### なぜ Assembled View 方式を採用するか

APM の `apm install` は現状 `.apm/instructions/` と `.apm/hooks/` を認識しない。これを変えるには APM 本体の改修が必要で本リポジトリの管轄外となる。しかし：

- Canonical Source ファイルを先に作ることで、APM が対応した際の移行コストがゼロになる
- CI での存在検証を追加することで「Canonical Source を書かずに agent.md だけ書く」という退行を防止できる
- `assembled view` であることを明記することで、編集者は「agent.md を直接編集してよい」という誤解を回避できる

代替案として「agent.md から instruction/hook セクションを完全に削除する」案も検討したが、APM が対応するまでの間エージェントが rules を参照できなくなるため不採用。

### なぜ `execution-policy.md` と `session-start.md` の2ファイルか

`[role: instruction]` タグの7セクションはすべて「実行ポリシー」（何をすべきか/すべきでないか）であり、単一の `execution-policy.md` にまとめることで参照パスが安定する。`[role: hook]` の Auto-Memory Protocol は発火イベント（SessionStart, PhaseGateComplete, 15-exchange）が複数あるが、memoir スキルの操作という単一の責務にまとめられるため `session-start.md` というイベント起点の名前で統一する（将来的には発火イベントごとにファイル分割してよい）。

### ADR-009 残 TODO へのアプローチ

テキスト類似度ベースのチェッカは実装コストが高く誤検知リスクもある。代わりに「ステータス語の存在検証」という構造的チェックを採用する: easy-agent の Fallback Chain に `| **Verify — ESCALATE**` と書いてあれば、`call-refine-loop/SKILL.md` の Caller Response Contract に `ESCALATE` という語が存在するかを確認する。意味の整合性は人間のレビューに委ねるが、語彙レベルのドリフト（ステータス名の変更漏れ等）は CI で捉えられる。

## 結果として得られる特性

- **責務分離の物理化**: アノテーションだけでなく、ファイルシステムレベルで役割の境界が明確になる
- **Canonical Source の一意性**: 編集箇所が明確になり「どこを変えればよいか」が自明
- **再利用の足がかり**: `instructions/execution-policy.md` は将来的に他エージェント（advisor, parliament-chairperson 等）からも参照可能になる
- **ADR-009 ドリフト抑制**: Fallback Chain と Caller Response Contract のステータス語彙の乖離を CI で検出
- **APM 移行準備**: APM が `instructions/` と `hooks/` をサポートした際に、Assembled View 解除のみで移行完了

## 今後の課題

- APM が `.apm/instructions/` / `.apm/hooks/` のインライン展開をサポートした際に、`easy-agent.agent.md` の assembled view セクションを削除して Canonical Source のみにする
- `easy-agent.agent.md` の Assembled View と Canonical Source のコンテンツドリフト検出（語彙レベル以上の意味整合性チェック）
- `instructions/execution-policy.md` の `applyTo:` フィールドによる他エージェントへのスコープ拡大（advisor, parliament-chairperson 等への適用）
- Hook イベント語彙の標準化（SessionStart / PhaseGateComplete / NExchangeElapsed の公式名称化）

## 関連決定

- ADR-010: 役割明確化タクソノミー — 本 ADR が実現する物理分離の概念的基盤
- ADR-009: Caller Response Contract Convention — Lint B (ステータス整合性検証) の対象規約
- ADR-005: APM パッケージング — `.apm/` ディレクトリ構造の拡張
