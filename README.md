# OWF — Outline-driven WorkFlow

**「仕事はアウトラインが9割」** をワークフローとして実装した Claude Code プラグインです。

3フェーズのシンプルな開発ワークフロー:

```
Phase 1: /owf:outline
  ├─ 1a. Outline 作成    (owf-outliner)
  └─ 1b. Critic ループ    (owf-outline-critic × 2 並列, max 3 往復)
Phase 2: /owf:implement
  └─ TDD 実装             (owf-implementer — プロジェクト言語を自動検出)
Phase 3: /owf:review      (デュアルモード、git diff から自動判別)
  ├─ Outline Review       (owf-outline-critic × 2 並列 — 単発・pr.md なし)
  └─ Implementation Review (/simplify → owf-reviewer × 2 並列 → owf-fixer loop max 3 → pr.md)
```

`/owf:review` は `git diff --name-only` に `outlines/**` 以外の変更ファイルがあるかで Outline Review / Implementation Review を自動判別します。

## インストール

```bash
# sc30gsw マーケットプレイスを追加
/plugin marketplace add sc30gsw/owf

# OWF プラグインをインストール
/plugin install owf@sc30gsw-owf

# プラグインreload
/reload-plugin
```

または `/plugin` コマンドで UI を開き、マーケットプレイスタブから操作することもできます。

インストール後、OWF のスキルとエージェントが利用可能になります。

---

## Skills（スラッシュコマンド）

Skills はオーケストレータとして動作し、内部でエージェントを呼び出してループを管理します。

### `/owf:outline` — Phase 1: Outline 作成

```
/owf:outline <タスク記述>
```

**何をするか**：タスク記述から `outline.md` を生成し、敵対的 critic ループで品質を担保します。内部は **1a（作成）→ 1b（レビュー）** の 2 ステップ構成です。

**1a. Outline 作成**（`owf-outliner`）
1. プロジェクト言語を自動検出（`OWF_LANG` 環境変数 > `README.md` サンプリング > デフォルト `en`）
2. `./outlines/<slug>/outline.md` を骨子テンプレートから作成
3. `owf-outliner` (opus, effort: xhigh) がコードベースを調査し outline を記述

**1b. Critic ループ**（`owf-outline-critic` × 2 並列）
4. `owf-outline-critic` を **2体並列**で起動
   - **Critic A**: `axis_focus: ["clarity", "decomposition"]`（構造観点、0–50点）
   - **Critic B**: `axis_focus: ["risk", "reuse"]`（文脈観点、0–50点）
5. orchestrator が 2 体の partial Verdict をマージ（Dimensions 結合、score 加算 → 0–100、Findings dedupe、band 算出）
6. バンド判定で分岐：
   - 🟢 GREEN (≥80) → 終了、`## Remaining Risks` を追記
   - 🟡 YELLOW (75–79) → ループ停止、`outline-pr.md` 生成 → ユーザー確認
   - 🔴 RED (<75) → **1a に戻り** `owf-outliner` が指摘セクションのみ修正（Mode B）→ **1b を再実行**
7. 最大 3 往復。上限到達時は `## Score Improvement Suggestions` を追記して停止

**スコア分岐**：
| スコア | バンド | アクション |
|---|---|---|
| 100 | Perfect | そのまま次フェーズへ |
| 80–99 | 🟢 GREEN | `## Remaining Risks` を追記して終了 |
| 75–79 | 🟡 YELLOW | ループ停止。`outline-pr.md` 生成 → ユーザー確認 |
| < 75 × 3回 | 🔴 RED | `## Score Improvement Suggestions` を追記して停止 |

**成果物**：
- `./outlines/<slug>/outline.md` — 実装フェーズの唯一の真実源
- `./outlines/<slug>/outline-pr-for-<slug>.md` — スコア < 100 の場合のみ生成

**例**：
```
/owf:outline ユーザーリスト画面の検索入力に useDebounce を追加する
/owf:outline Add rate limiting middleware to the API gateway
```

---

### `/owf:implement` — Phase 2: TDD 実装

```
/owf:implement ./outlines/<slug>/outline.md
```

**何をするか**：`outline.md` を唯一の真実源として TDD で実装します。outline は絶対に変更しません。

**フロー**：
1. `outline.md` を検証（必須セクションの存在確認）
2. ステップ間の依存関係を分析し、並列実行可否を判断
3. `owf-implementer` (sonnet) を起動（独立 feature があれば Agent Teams で並列化）
4. テスト全 pass まで自走 → 型チェック → lint
5. 完了後「`/owf:review` を実行してください」と案内

**重要な制約**：
- `outline.md` は実装中に変更禁止（read-only として扱う）
- outline と実装の乖離を検知した場合は即座に停止してユーザーに報告

**成果物**：実装コード + テストファイル（outline で指定されたファイル群）

---

### `/owf:review` — Phase 3: レビュー（デュアルモード）

```
/owf:review ./outlines/<slug>/outline.md
```

**何をするか**：2 つの明確に分離されたモードで動作します。

| モード | 対象 | 使用エージェント | 特徴 |
|---|---|---|---|
| **Outline Review** | `outline.md` 単体の品質確認 | `owf-outline-critic` × 2 並列 | 敵対的のみ（`/simplify` なし、fixer なし、単発・ループなし） |
| **Implementation Review** | 実装コードの `outline.md` への整合性・コード品質 | `owf-reviewer` × 2 並列 + `owf-fixer` | 敵対的 + `/simplify` 前処理 + 修正ループ（最大3回） |

**モード自動判別**：`git diff --name-only` の結果に `outlines/**` 以外の変更ファイルがあれば Implementation Review、なければ Outline Review。

**Outline Review フロー**：
1. `outline.md` を読み込む
2. `owf-outline-critic` を **2体並列**で起動（Critic A: `clarity + decomposition`、Critic B: `risk + reuse`）
3. 両方の partial Verdict をマージ（スコア加算・findings 統合・band 算出）
4. CLI に Score / Band / 根拠 / 指摘事項 / 次アクション を表示（pr.md 生成なし、ループなし）

**Implementation Review フロー**：
1. `outline.md` と `git diff` を収集
2. **`/simplify` skill を実行**（初回のみ・`git diff` 対象ファイルに限定）— 機械的な整理を事前に済ませ、reviewer の判定精度を高める
3. `owf-reviewer` を **2体並列**で起動（Reviewer A: `fidelity + tests`、Reviewer B: `simplify + maintain`）
4. 両方の partial Verdict をマージ（スコア加算・findings 統合・band 算出）
5. `owf-fixer` が RED 指摘を修正 → 再レビュー（最大3回）
6. `./outlines/<slug>/pr.md` を生成
7. CLI に Score / Band / 根拠 / 次アクション を表示

**Implementation Review CLI 出力例**：
```
========================================
OWF Implementation Review: ./outlines/add-debounce/outline.md
========================================
スコア: 87 / 100
バンド: 🟢 GREEN
イテレーション: 2 / 3
----------------------------------------
評価の根拠:
  fidelity   23/25: outline 全項目と整合
  tests      22/25: coverage 83%
  simplify   21/25: 重複 util を統合済み
  maintain   21/25: 軽微: user-list.tsx 340行
----------------------------------------
次のアクション:
  1. pr.md を確認: ./outlines/add-debounce/pr.md
  2. gh pr create --body-file ./outlines/add-debounce/pr.md
========================================
```

**Outline Review CLI 出力例**（単発・pr.md なし）：
```
========================================
OWF Outline Review: ./outlines/add-debounce/outline.md
========================================
スコア: 82 / 100
バンド: 🟢 GREEN
----------------------------------------
評価の根拠:
  clarity        22/25: 完了条件が一文で読める
  decomposition  20/25: ステップが RED→GREEN→REFACTOR で分解済み
  risk           20/25: エッジケース列挙あり、ロールバック明記
  reuse          20/25: 既存の useDebounce 参照あり
----------------------------------------
指摘事項:
  [MED] ## Verification: 手動確認手順が抽象的 → 具体的な画面操作を記述
  [LOW] ## Out of Scope: 英語と日本語が混在
----------------------------------------
次のアクション:
  このまま `/owf:implement` に進めます。
========================================
```

**成果物**：`./outlines/<slug>/pr.md` — GitHub PR 作成用テンプレート

---

### `/owf:rubric` — スコアリング詳細表示

```
/owf:rubric
```

**何をするか**：OWF のスコアリング規約（4軸×25点、RYG バンド、Verdict フォーマット）を表示します。レビュー結果の根拠を確認したい時に使います。

---

## Agents（エージェント）

各エージェントは `@agent名` で単独起動することもできます。スキルのループを途中でスキップしたり、手動介入したい場合に便利です。

### `@owf-outliner` — Outline 生成・修正役

| 属性 | 値 |
|---|---|
| モデル | opus (effort: xhigh) |
| ツール | Read, Write, Edit, Grep, Glob, Bash |
| フェーズ | Phase 1 |

**役割**：タスク記述からコードベースを調査し `outline.md` を記述します。critic からフィードバックを受けた場合は指摘されたセクションのみを修正します。

**動作モード**：
- **Mode A（初回作成）**：コードベースの既存実装・ユーティリティを検索し、再利用できるものを `## Existing Code to Reuse` に記載してから全セクションを埋める
- **Mode B（修正）**：Findings で指定されたセクションのみ修正。他のセクションは変更しない

**特徴**：
- 検出言語（`lang=ja/en`）に応じてセクションヘッダーも含め全テキストを適切な言語で記述
- スコープ外の拡張は行わない。outline に書かれていない機能は提案のみ（`## Risks` に記載）

---

### `@owf-outline-critic` — Outline 敵対的レビュアー

| 属性 | 値 |
|---|---|
| モデル | opus (effort: xhigh) |
| ツール | **Read, Grep, Glob のみ（Write/Edit なし）** |
| フェーズ | Phase 1 / Phase 3 Outline Review モード |

**役割**：`outline.md` の弱点・曖昧さ・欠落を徹底的に洗い出します。直接ファイルを編集する手段を物理的に持ちません（frontmatter の `tools:` で強制）。`/owf:outline` および `/owf:review`（Outline Review モード）からは **2体並列で起動**されます。

**並列起動と axis focus**：
- **Critic A** (`axis_focus: ["clarity", "decomposition"]`) — 構造観点。スコア 0–50。
- **Critic B** (`axis_focus: ["risk", "reuse"]`) — 文脈観点。スコア 0–50。
- orchestrator が両方の partial Verdict をマージして最終スコア（0–100）を算出。
- `@owf-outline-critic` として単体起動する場合は `axis_focus` なしで全4軸（0–100）を採点。

**採点軸（各 25 点）**：
- `clarity` — ゴール・スコープ・完了条件が一文で読めるか
- `decomposition` — ステップが適切粒度で、独立テスト可能か
- `risk` — 前提・エッジケース・ロールバックが明記されているか
- `reuse` — 既存コードを参照し NIH していないか

**厳格ルール**：
- `axis_focus` あり: 初回レビューで担当2軸の合計 42/50 を超えない
- `axis_focus` なし: 初回レビューで 85/100 を超えない
- すべての指摘には具体的なセクション名または行番号が必須
- 「全体的に良い」など曖昧な肯定はしない

**応答フォーマット**（Verdict ブロック固定、これ以外のテキストを含まない）：
```
### Verdict
score: 72 / band: RED / iteration: 1/3

### Dimensions
- clarity: 20/25 — ...
...
### Findings
- [HIGH] ## ゴール: ...
### Routing
to: owf-outliner / sections: [...]
```

---

### `@owf-implementer` — TDD 実装担当

| 属性 | 値 |
|---|---|
| モデル | sonnet |
| ツール | Read, Write, Edit, Bash, Grep, Glob |
| フェーズ | Phase 2 |

**役割**：`outline.md` に従い RED→GREEN→REFACTOR サイクルで実装します。TDD ロジックを自身のプロンプトに内包しており、外部の tdd-guide agent には依存しません。

**言語・規約の自動検出（TypeScript を前提としない）**：

実装開始前に必ずプロジェクトの言語と規約を検出します。

1. **言語判定**（manifest ファイルで first-match-wins）
   - `package.json` → TS/JS、`Cargo.toml` → Rust、`go.mod` → Go、`pyproject.toml` / `requirements.txt` → Python、`pom.xml` / `build.gradle*` → Java/Kotlin、`Gemfile` → Ruby、`composer.json` → PHP、`*.csproj` → C#、`Package.swift` → Swift、`mix.exs` → Elixir、など
2. **規約ソースを読み込み**（存在するものすべて）
   - `./CLAUDE.md`、`./.claude/CLAUDE.md`
   - `./.claude/rules/**/*.md`（プロジェクトスコープのルール）
   - `~/.claude/rules/common/*.md` + `~/.claude/rules/<lang>/*.md`（ユーザースコープ、`everything-claude-code` 形式の rules がインストールされていれば参照）
   - 周辺ファイル（命名・エラーハンドリング・型定義の既存パターン）
3. **優先順位**（特定度が高い方が勝つ）
   outline.md > プロジェクト CLAUDE.md > プロジェクト `.claude/rules/` > ユーザースコープ `<lang>/` > ユーザースコープ `common/` > 言語イディオムのデフォルト

**コード品質 — 言語非依存の原則（常に適用）**：

以下は普遍原則で、言語固有のイディオムと衝突した場合はイディオム側が勝ちます（例：Go のポインタレシーバ、Rust の `Result<T, E>`、Python の `raise`/`except`）。

- **KISS / DRY / YAGNI** — 最小の解、実在する重複のみ抽出、必要になるまで作らない
- **ファイルサイズ**：目安 200–400行、上限 800行（プロジェクトの CLAUDE.md で上書き可）
- **関数サイズ**：目安 <50行、ネスト深さ ≤4 レベル（早期 return を活用）
- **型/モデルの SSoT**：重複定義せず言語の機能で派生（TS `Pick`/`Omit`、Python `TypedDict`、Rust 構造体再利用、など）
- **不変性をデフォルトに**：言語イディオムが許す範囲で（Go のポインタレシーバなどは除外）
- **明示的なエラーハンドリング**：言語の慣用形に従う — 例外（Python/Java）、error 返却（Go）、`Result<T, E>`（Rust）、プロジェクトの既存パターン（TS）。**エラーを握り潰さない**
- **境界での入力検証**：外部入力（ユーザー、API、ファイル）をスキーマベースで検証（Zod、Pydantic、validator、Bean Validation など）
- **本番コードにデバッグ出力を残さない**：`console.log` / `println!` / `print()` / `System.out.println` を禁止、プロジェクトのロガーを使用
- **シークレットをハードコードしない**：環境変数 or シークレットマネージャーのみ
- **命名は言語慣用に従う**：周辺ファイルの既存パターンに一致させる（TS 変数 camelCase、Rust/Python snake_case、Go/TS 型 PascalCase、TS/JS ファイル名は既存プロジェクトの規約を踏襲）

**完了条件**：プロジェクトのツールチェーンで テスト全 pass + 静的/型チェッククリーン + カバレッジ ≥ 80%（プロジェクトが別閾値を指定していればそれに従う）

---

### `@owf-reviewer` — 実装 敵対的レビュアー

| 属性 | 値 |
|---|---|
| モデル | opus (effort: xhigh) |
| ツール | **Read, Grep, Glob, Bash（read-only コマンドのみ）** |
| フェーズ | Phase 3 |

**役割**：実装が `outline.md` と 1:1 で整合しているかを徹底検証します。Write/Edit ツールを持たないため物理的にコードを変更できません。`/owf:review` からは **2体並列で起動**されます。

**並列起動とaxis focus**：
- **Reviewer A** (`axis_focus: ["fidelity", "tests"]`) — 正しさ観点。スコア 0–50。
- **Reviewer B** (`axis_focus: ["simplify", "maintain"]`) — 品質観点。スコア 0–50。
- orchestrator が両方の partial Verdict をマージして最終スコア（0–100）を算出。
- `@owf-reviewer` として単体起動する場合は `axis_focus` なしで全4軸（0–100）を採点。

**採点軸（各 25 点・言語非依存）**：
- `fidelity` — outline の全完了条件が実装されているか、スコープ逸脱はないか
- `tests` — TDD が守られているか、全テストが pass するか、カバレッジ ≥ 80% か
- `simplify` — 再利用性・品質・効率（重複コード・ファイルサイズ・ネスト深さ・不変性・**プロジェクト既存のエラーハンドリング慣用形との整合**・デバッグ出力の残留・未使用コード）
- `maintain` — 命名・ファイル分割・コーディング規約（**プロジェクトの言語慣用に従った命名**、型/モデル SSoT、境界での入力バリデーション）

> 採点時は `owf-implementer` と同じ規約ソース（プロジェクト `CLAUDE.md` / `.claude/rules/` / `~/.claude/rules/<lang>/`）を参照し、プロジェクトが採用していないパターン（例: `Result<T, E>` を使わないプロジェクトでの強制）は減点しません。

**厳格ルール**：
- 完了条件が1つでも未実装: `fidelity` ≤ 15/25 確定
- カバレッジ < 80%: `tests` ≤ 10/25 確定
- `axis_focus` あり: 初回レビューで担当2軸の合計 42/50 を超えない
- `axis_focus` なし: 初回レビューで 85/100 を超えない

---

### `@owf-fixer` — 実装 修正担当

| 属性 | 値 |
|---|---|
| モデル | sonnet |
| ツール | Read, Write, Edit, Bash, Grep, Glob |
| フェーズ | Phase 3 |

**役割**：`owf-reviewer` の Findings を受け取り、指摘箇所に対して最小限の修正を適用します。

**修正優先度**：
- `[HIGH]` — 必ず修正（outline との整合性・テスト品質に関わるもの）
- `[MED]` — 基本修正（スコープ拡張が不要なもの）
- `[LOW]` — 軽微なら修正、スキップ時は理由を報告

**simplify 観点を修正時に適用**（2回目以降のイテレーション、言語非依存）：
- 修正するファイルで見つけた重複コード・型定義をついでに整理
- プロジェクトの上限を超えたファイルを分割（デフォルト 800 行、CLAUDE.md で上書き可）
- 触れたコードのエラーハンドリングを**プロジェクト既存の慣用形**に揃える（try-catch ↔ Result ↔ error 返却などを**勝手に**切り替えない）
- 触れたコードのデバッグ出力（`console.log` / `println!` / `print()` / `System.out.println` 等）を除去
- ※ 初回（iter=0）は `/simplify` skill による前処理が実施済みのため、機械的整理は既に完了

**言語検出と規約ソース**：`owf-implementer` と同じ手順（manifest から言語判定 → `CLAUDE.md` / `.claude/rules/` / `~/.claude/rules/<lang>/` を参照）でプロジェクト規約に従います。

**完了条件**：プロジェクトのツールチェーンで 修正後に全テスト pass + 静的/型チェッククリーン を確認してから報告

---

## エージェント直接利用ワークフロー（skill を使わず `@agent` だけで運用する）

スキル（`/owf:*`）のオーケストレーションを使わず、各エージェントを `@` プレフィックスで**直接**呼び出して手動で 3 フェーズを回すこともできます。ループの途中にレビュアーを挟みたい、1 フェーズだけ再実行したい、CI から個別に起動したい、などのユースケース向けです。

### Phase 1 — Outline 作成＋critic を手動で回す

```
# 1a. 初回の outline を書かせる
@owf-outliner
task: <タスク記述>
lang: ja   # 省略可。未指定時は owf-outliner が検出
write_to: ./outlines/<slug>/outline.md

# 1b. critic を 2 体並列で起動（1 つのメッセージ内に 2 ブロック）
@owf-outline-critic
axis_focus: ["clarity", "decomposition"]
iteration: 1/3
outline: ./outlines/<slug>/outline.md

@owf-outline-critic
axis_focus: ["risk", "reuse"]
iteration: 1/3
outline: ./outlines/<slug>/outline.md

# → 2 体の partial Verdict（各 0–50）を自分でマージして 0–100 に合算
# → RED (<75) なら Findings を渡して owf-outliner を Mode B で再起動
@owf-outliner
mode: revision
findings: <critic Verdict の Findings を貼り付け>
sections: <Routing の sections を貼り付け>
outline: ./outlines/<slug>/outline.md
```

### Phase 2 — 実装を直接起動

```
@owf-implementer
outline: ./outlines/<slug>/outline.md
# owf-implementer がプロジェクト言語・規約を自動検出して TDD を回す
```

独立 feature が複数あれば、同じ 1 メッセージで `@owf-implementer` を複数ブロック並列起動し、各ブロックで担当ステップ範囲を指定します。

### Phase 3 — レビュー＋修正を手動ループ

```
# Outline Review だけしたい（実装 diff がない場合）
@owf-outline-critic
axis_focus: ["clarity", "decomposition"]
outline: ./outlines/<slug>/outline.md

@owf-outline-critic
axis_focus: ["risk", "reuse"]
outline: ./outlines/<slug>/outline.md

# Implementation Review（実装 diff がある場合）— 2 体並列 reviewer
@owf-reviewer
axis_focus: ["fidelity", "tests"]
iteration: 1/3
outline: ./outlines/<slug>/outline.md
diff: <git diff の結果>

@owf-reviewer
axis_focus: ["simplify", "maintain"]
iteration: 1/3
outline: ./outlines/<slug>/outline.md
diff: <git diff の結果>

# → マージ後 RED なら fixer を起動
@owf-fixer
findings: <merged Verdict の Findings>
files: <Routing の files 和集合>
# → 完了後に reviewer 2 体並列を再実行（max 3 往復）
```

### skill と直接利用の比較

| 観点 | `/owf:*` skill 経由 | `@owf-*` 直接利用 |
|---|---|---|
| オーケストレーション | 自動（並列 spawn / Verdict マージ / ループ判定 / pr.md 生成） | 手動 |
| `/simplify` 前処理 | Phase 3 Implementation Review の初回で自動実行 | 手動で起動する必要あり |
| スコアマージ | orchestrator が自動で 0–50 × 2 → 0–100 計算 | 手動で合算（Dimensions 結合 / Findings dedupe） |
| ループ制御 | 最大3イテレーションの自動管理 | 自分で反復判定 |
| 向いている用途 | 通常の開発フロー | CI 連携 / 1 フェーズだけ再実行 / 途中介入 / デバッグ |

両者は併用可能です（例：skill で Phase 1–2 を回して、Phase 3 だけ手動 `@owf-reviewer` で細かく制御）。

---

## スコアリング

両フェーズ共通の採点基準。詳細は `/owf:rubric` で確認。

| スコア | バンド | アクション |
|---|---|---|
| 100 | Perfect | 指摘なし。次フェーズへ |
| 80–99 | 🟢 GREEN | 採用。残リスク/未充足点を成果物末尾に追記 |
| 75–79 | 🟡 YELLOW | ループ即停止。PR ノートを生成してユーザー確認 |
| < 75 | 🔴 RED | 次イテレーションへ（最大3回） |
| < 75 × 3回 | 🔴 RED 上限 | 改善提案を追記して停止 |

---

## 設定

### 環境変数

| 変数 | デフォルト | 説明 |
|---|---|---|
| `OWF_LANG` | 自動検出 | outline の記述言語（`ja`, `en`, `zh` など）。未設定時は README.md をサンプリングして判定 |
| `OWF_MAX_ITER_OUTLINE` | `3` | Phase 1 critic ループの最大往復回数 |
| `OWF_MAX_ITER_REVIEW` | `3` | Phase 3 reviewer/fixer ループの最大往復回数 |
| `OWF_TRACE` | 無効 | `1` に設定すると `./outlines/<slug>/.owf-trace.log` にスコアログを1行 append |

### `.owf-config.json`（プロジェクトルート）

プロジェクト固有の設定をファイルで管理したい場合:

```json
{
  "lang": "ja",
  "maxIterOutline": 2,
  "maxIterReview": 3
}
```

---

## ファイル構成

```
owf/
├── .claude-plugin/
│   ├── plugin.json          # プラグイン定義
│   └── marketplace.json     # マーケットプレイス公開用
├── CLAUDE.md                # プラグイン内部ガイド
├── README.md                # このファイル
├── skills/
│   ├── owf-rubric/SKILL.md  # /owf:rubric — スコアリング詳細
│   ├── owf-outline/SKILL.md # /owf:outline — Phase 1 オーケストレータ
│   ├── owf-implement/SKILL.md # /owf:implement — Phase 2 オーケストレータ
│   └── owf-review/SKILL.md  # /owf:review — Phase 3 オーケストレータ
├── agents/
│   ├── owf-outliner.md          # sonnet, Write可
│   ├── owf-outline-critic.md    # opus, effort:xhigh, READ-ONLY
│   ├── owf-implementer.md       # sonnet, TDD内包
│   ├── owf-reviewer.md          # opus, effort:xhigh, READ-ONLY
│   └── owf-fixer.md             # sonnet, simplify観点内包
└── templates/
    ├── outline.md.tmpl          # Outline 骨子
    ├── outline-pr.md.tmpl       # Phase 1 PR ノート
    └── pr.md.tmpl               # Phase 3 最終 PR テンプレート
```

---

## 外部依存

**Claude Code 組み込み `simplify` skill のみ。** Phase 3 (`/owf:review`) の冒頭で、グローバルスコープに存在する `simplify` skill を呼び出します。このスキルは Claude Code に標準搭載されており、追加インストールは不要です。他の外部プラグイン（everything-claude-code 等）への依存はありません。
