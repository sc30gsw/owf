# OWF — Outline-driven WorkFlow

**「仕事はアウトラインが9割」** をワークフローとして実装した Claude Code プラグインです。

3フェーズのシンプルな開発ワークフロー:

```
Phase 1: /owf:outline  → outline.md (adversarial critic loop × max 3)
Phase 2: /owf:implement → TDD 実装
Phase 3: /owf:review   → adversarial reviewer/fixer loop × max 3 → pr.md
```

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

**何をするか**：タスク記述から `outline.md` を生成し、敵対的 critic ループで品質を担保します。

**フロー**：
1. プロジェクト言語を自動検出（`OWF_LANG` 環境変数 > `README.md` サンプリング > デフォルト `en`）
2. `./outlines/<slug>/outline.md` を骨子テンプレートから作成
3. `owf-outliner` (opus, effort: xhigh) がコードベースを調査し outline を記述
4. `owf-outline-critic` (opus, effort: xhigh) が敵対的にレビューし Verdict を返却
5. スコアに応じてループ継続または終了（最大3往復）

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

### `/owf:review` — Phase 3: 実装レビュー

```
/owf:review ./outlines/<slug>/outline.md
```

**何をするか**：実装を `outline.md` との整合性・コード品質・テスト品質の観点で adversarial にレビューし、`pr.md` を生成します。

**フロー**：
1. `outline.md` と `git diff` を収集
2. Agent Teams で `owf-reviewer` (READ-ONLY) と `owf-fixer` を起動
3. reviewer → fixer の往復ループ（最大3回）
4. `./outlines/<slug>/pr.md` を生成
5. CLI に Score / Band / 根拠 / 次アクション を表示

**CLI 最終出力例**：
```
========================================
OWF Review: ./outlines/add-debounce/outline.md
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
| フェーズ | Phase 1 |

**役割**：`outline.md` の弱点・曖昧さ・欠落を徹底的に洗い出します。直接ファイルを編集する手段を物理的に持ちません（frontmatter の `tools:` で強制）。

**採点軸（各 25 点）**：
- `clarity` — ゴール・スコープ・完了条件が一文で読めるか
- `decomposition` — ステップが適切粒度で、独立テスト可能か
- `risk` — 前提・エッジケース・ロールバックが明記されているか
- `reuse` — 既存コードを参照し NIH していないか

**厳格ルール**：
- 初回レビューでは絶対に 85/100 を超えない
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

**実装規約（常に適用）**：
- TypeScript: `Pick<T>` / `Omit<T>` / `Record<K,V>` を使い型の重複定義を避ける
- エラー処理: `Result<T, E>` パターン（try-catch 禁止）
- 不変性: オブジェクトを直接変更せず新しいオブジェクトを返す
- ファイルサイズ: 200–400行が目安、800行を超えたら分割
- ファイル命名: ケバブケース（`user-card.tsx`、`use-debounce.ts`）
- `console.log` / ハードコードされたシークレット禁止

**完了条件**：テスト全 pass + 型チェッククリーン + カバレッジ ≥ 80%

---

### `@owf-reviewer` — 実装 敵対的レビュアー

| 属性 | 値 |
|---|---|
| モデル | opus (effort: xhigh) |
| ツール | **Read, Grep, Glob, Bash（read-only コマンドのみ）** |
| フェーズ | Phase 3 |

**役割**：実装が `outline.md` と 1:1 で整合しているかを徹底検証します。Write/Edit ツールを持たないため物理的にコードを変更できません。

**採点軸（各 25 点）**：
- `fidelity` — outline の全完了条件が実装されているか、スコープ逸脱はないか
- `tests` — TDD が守られているか、全テストが pass するか、カバレッジ ≥ 80% か
- `simplify` — 再利用性・品質・効率（重複コード、ファイルサイズ、不変性、Result パターン、未使用 import）
- `maintain` — 命名・ファイル分割・コーディング規約（immutability、型定義 SSoT、入力バリデーション）

**厳格ルール**：
- 完了条件が1つでも未実装: `fidelity` ≤ 15/25 確定
- カバレッジ < 80%: `tests` ≤ 10/25 確定
- 初回レビューでは 85/100 を超えない

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

**simplify 観点を修正時に適用**：
- 修正するファイルで見つけた重複コード・型定義をついでに整理
- 800行を超えたファイルは分割
- 触れたコードの try-catch を Result パターンに変換

**完了条件**：修正後に全テスト pass + 型チェッククリーン を確認してから報告

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

**なし。** このプラグインは Claude Code 組み込みツール（Read/Write/Edit/Bash/Task/TeamCreate）のみに依存します。他プラグイン（everything-claude-code、code-simplifier 等）は不要です。どのプロジェクトでも動作します。
