# ADR-024: APM Manifest & Agent Frontmatter Integrity CI 保護

## ステータス

Accepted

## コンテキスト

CLAUDE.md には APM パッケージに関する 2 つの明示的な不変条件が記載されている。

1. **バージョン一致**: 「ルート `apm.yml` と全サブパッケージの `<package>/apm.yml` は同一の `version:` を共有し、同時にバンプする」
2. **`type: hybrid` 必須**: 「`apm.yml` の `type` フィールドは必ず `hybrid` にする」

また、エージェント定義ファイル (`*.agent.md`) にはフロントマターに `name:`, `description:`, `model:`, `tools:` が必要であることが慣習として存在するが、文書化も CI 強制もされていない。

これら 3 つの不変条件はいずれも CI による保護を持たないため、以下のサイレントな退行が検知されない：

1. **バージョンドリフト**: 新規パッケージ追加時や `Release - Prepare` ワークフロー外での手動バンプにより、`apm.yml` 間のバージョンが乖離しても CI は検知しない。
2. **`type` フィールド欠落**: 新規パッケージの `apm.yml` で `type: hybrid` を忘れても CI は検知しない。`apm install` の挙動がパッケージ間で非一貫になる。
3. **Frontmatter 欠落**: `*.agent.md` に `name:`, `description:`, `model:`, `tools:` のいずれかが存在しないエージェントが追加されても CI は検知しない。Lint I/J（ADR-022）が依存する `name:` frontmatter が欠落した場合、エージェント名参照検証が無音でスキップされる。

## 決定

**以下の 3 つの CI Lint を `check.yml` に追加する。**

### Lint N — apm.yml version uniformity

すべての `*/apm.yml`（サブパッケージ）とルートの `apm.yml` が同一の `version:` 値を宣言していることを検証する。

- ルート `apm.yml` の `version:` を基準値として読み込む
- 全サブパッケージの `*/apm.yml` を走査し、基準値と異なる `version:` があれば失敗

### Lint O — apm.yml `type: hybrid` 必須

すべての `*/apm.yml`（サブパッケージ）が `type: hybrid` を宣言していることを検証する。

- `type:` フィールドが存在しないか `hybrid` 以外の値を持つ場合に失敗
- ルート `apm.yml` は `type:` を宣言しないケースがあるため対象外とする

### Lint P — agent frontmatter 必須フィールド

すべての `*/.apm/agents/*.agent.md` のフロントマターに `name:`, `description:`, `model:`, `tools:` の 4 フィールドが存在することを検証する。

- フロントマターは `---` で区切られた YAML ブロックとして解析
- 4 フィールドのいずれかが欠落している場合に失敗
- Lint I/J（ADR-022）の前提条件（`name:` 存在）を明示的に保護する副次効果がある

## 設計の根拠

### なぜ「今」か

ADR-021（Canonical Source Section Registry）と ADR-022（Agent Invocation Name Coverage）は「ADR 決定 + 対応 CI Lint を同時追加」するパターンを確立した。本 ADR はその後継として、CLAUDE.md に明文化されているが CI 未保護の不変条件を遡及的にカバーする。これは ADR-023（is_critical / residual_risks CI 保護）が採用したアプローチと同一である。

### なぜ Lint N/O を別ステップに分けるか

- Lint N はバージョン乖離を検出し「誰かが手動バンプした」という事実を明確にする
- Lint O は `type:` フィールドの有無・値を検出し「パッケージ設定のミス」を明確にする

エラーメッセージを混在させると診断が困難になるため、ADR-022 の方針（パターンごとに別ステップ）に準じて分割する。

### なぜ Lint P でルート `apm.yml` は対象外か

ルートの `apm.yml` はパッケージ一覧（`easy-agent`）のみを管理するメタマニフェストであり、`type:` フィールドをもともと持たない設計になっている。サブパッケージとは役割が異なるため、Lint O の対象から外す。

## 結果

| 不変条件 | CLAUDE.md 記載 | CI 保護前 | CI 保護後 |
| :--- | :--- | :--- | :--- |
| 全 `apm.yml` の `version:` 一致 | ✅ 明記 | ❌ 未保護 | ✅ Lint N |
| 全サブパッケージ `apm.yml` の `type: hybrid` | ✅ 明記 | ❌ 未保護 | ✅ Lint O |
| 全エージェントの frontmatter 必須フィールド | 慣習 | ❌ 未保護 | ✅ Lint P |

### 既存 Lint との整合

- **Lint I / J (ADR-022)**: `name:` frontmatter が存在することを前提に全ファイルをスキャンする。Lint P が `name:` の存在を保証することで、Lint I/J の前提条件が CI レベルで担保される。
- **Lint C (ADR index)**: ADR-024 本文書を `docs/adr/README.md` に追加することで Lint C が継続 pass する。

### 影響範囲

- 追加ファイル: `docs/adr/ADR-024-apm-manifest-integrity.md`（本文書）
- 変更ファイル: `.github/workflows/check.yml`（Lint N / O / P の 3 ステップ追加）、`docs/adr/README.md`（ADR-024 エントリー追加）
- 変更なし: `apm.yml` ファイル群、エージェントファイル群（既に全制約を満たしている）

## 関連 ADR

- [ADR-005](./ADR-005-apm-packaging.md) — APM によるパッケージ管理（`type: hybrid` と version 管理の原点）
- [ADR-021](./ADR-021-canonical-source-section-registry.md) — Canonical Source Section Registry（CI 同時追加パターンの先例）
- [ADR-022](./ADR-022-agent-invocation-name-coverage.md) — Agent Invocation Name Coverage（Lint P の保護対象 `name:` を使用する Lint I/J の定義元）
- [ADR-023](./ADR-023-critical-quality-model-ci-coverage.md) — CI Lint Coverage for [critical] Quality Model（遡及的 CI 追加パターンの先例）
