# ADR-021: Canonical Source Section Registry Protocol (カノニカルソース・セクション登録プロトコル)

## ステータス

Accepted

## コンテキスト

ADR-012 (物理ファイル分離) は `easy-agent.agent.md` を "Assembled View" として、
`easy-agent/.apm/instructions/execution-policy.md` と `easy-agent/.apm/hooks/session-start.md`
をカノニカルソースと位置付けた。`easy-agent.agent.md` には「本ファイルを直接編集しても
Canonical Source と乖離した場合は **CI が検出する**」と記されている。

しかし ADR-012 の実装時点で追加された CI (Lint A) は
**カノニカルソースファイルが存在するかどうか** のみを確認しており、
以下の状況を検出できなかった：

1. カノニカルソースのセクションを `easy-agent.agent.md` から削除しても CI が気づかない
2. `easy-agent.agent.md` に新たな `[role: instruction]` H2 を追加してもカノニカルソース側が更新されていないことを CI が気づかない
3. セクション名のリネームでアセンブルビューとカノニカルソースが乖離しても CI が気づかない

つまり ADR-012 が約束した「CI が検出する」は語彙レベルの乖離しか捕捉できていなかった。

## 決定

### カノニカルソース YAML フロントマター形式の標準化

`.apm/instructions/*.md` および `.apm/hooks/*.md` に **YAML フロントマター** を導入し、
そのファイルが提供するセクションとアセンブル先ファイルを機械可読な形式で宣言する。

```yaml
---
canonical_source:
  role: instruction          # または hook
  assembles_into: agents/easy-agent.agent.md   # .apm/ ディレクトリ相対パス
  sections:
    - "セクション名 (H2 見出し文字列と完全一致)"
    - "..."
---
```

フロントマターがないファイルはスコープ外として静かにスキップする。
これにより ADR-012 のスコープ (easy-agent のみ) と互換性を保ちながら、
将来他のパッケージに拡大する場合もフロントマターを追加するだけでよい。

### CI Lint G: 双方向セクション整合性確認

`check.yml` に **"Lint canonical source section registry (ADR-021 Lint G)"** ステップを追加する。

#### Forward check (カノニカルソース → アセンブルビュー)

`canonical_source.sections` に列挙したセクション名が
`assembles_into` ターゲットファイルに `## <section>` H2 見出しとして存在することを確認する。

#### Reverse check (アセンブルビュー → カノニカルソース)

`Assembled View` マーカーを持つ `*.agent.md` 内の全 `## ... [role: instruction]` および
`## ... [role: hook]` H2 見出しが、いずれかのカノニカルソースの `sections` リストで
網羅されていることを確認する。

双方向の確認により、次の操作が CI で検出される：

| 操作 | 検出方向 | 具体的なエラー |
| :--- | :--- | :--- |
| sections リストにあるセクションを agent.md から削除 | Forward | `section 'X' not found as H2 heading in agent.md` |
| agent.md に `[role: instruction]` H2 を追加し sections を更新しない | Reverse | `H2 'X' [role: instruction] not covered by any canonical source` |
| sections リストのセクション名をリネームし agent.md の H2 を更新しない | Forward + Reverse | 両方同時にエラー |
| agent.md の H2 をリネームし sections を更���しない | Reverse | `H2 'new name' not covered` + forward check for old name fails |

### 編集手順の更新

セクション名を変更する場合の手順：

1. カノニカルソースファイル (`.apm/instructions/*.md` or `.apm/hooks/*.md`) の H2 見出しを更新
2. 同ファイルのフロントマター `canonical_source.sections` リストを更新
3. アセンブルビュー (`easy-agent.agent.md`) の対応 H2 見出しを更新
4. `git commit` → Lint G が全件グリーンになることを確認

## 設計の根拠

### なぜ YAML フロントマターか

HTML コメントマーカー (`<!-- CS-BEGIN: ... -->`) と比較して：

- **YAML フロントマター**: 標準的な Markdown/APM 形式。Python `yaml` モジュールで解析可能。
  ファイルの先頭に集約されているため見落としにくい。APM の将来的なビルド対応も容易。
- **HTML コメント**: Markdown ではインライン配置となり境界が不明瞭。構造化解析が難しい。

### なぜ双方向チェックか

ADR-012 が "assembled view → canonical source" の単方向チェック (ファイル存在) のみだったことが
今回の問題の本質。"canonical source → assembled view" 方向のチェック (Forward) だけでも
セクション削除は検出できるが、セクション追加の漏れは検出できない。双方向にすることで
「registry に書いたことが実現されているか」と「実現されていることが registry に記録されているか」
の両方を保証できる。

### スコープの設計

フロントマターを持つファイルのみを対象とし、parliament / taskforce / advisor / refine-loop などの
パッケージは引き続き "直書き canonical source" として扱う (ADR-012 §3 の設計を維持)。
これらのパッケージへの拡大は将来の ADR に委ねる。

## 結果

| パッケージ | 状態 | 方向 | 説明 |
| :--- | :--- | :--- | :--- |
| `easy-agent` | ✅ 双方向保護 | Forward + Reverse | `execution-policy.md` / `session-start.md` にフロントマター追加 |
| その他 | — スコープ外 | — | フロントマターなし → Lint G はスキップ |

Lint G により ADR-012 が約束していた「CI が検出する」が **実際に動作する保証** となった。

## 今後の課題

- APM が `.apm/instructions/` のインライン展開をサポートした際に、
  `canonical_source.assembles_into` を APM のビルドターゲット宣言として活用できる
- `sections` に加えて `required_headings` (H3 以下の必須見出し) を宣言できるよう拡張可能
- parliament / taskforce など他パッケージへの適用 (将来 ADR)

## 関連 ADR

- [ADR-010](./ADR-010-role-taxonomy.md) — 役割タクソノミー (`[role: ...]` タグ体系)
- [ADR-011](./ADR-011-hook-specification-format.md) — Hook Specification Format
- [ADR-012](./ADR-012-physical-role-separation.md) — 物理ファイル分離 (本 ADR の前提 ADR)
