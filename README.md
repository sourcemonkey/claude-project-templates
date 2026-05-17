# Rails CLAUDE.md スタータキット

Claude Code に Rails 中規模プロジェクトを生成させるための CLAUDE.md 一式。

## 構成

```
.
├── CLAUDE.md                    ← チーム共通ルール（このリポジトリ配下全体に適用）
├── team-rules/
│   ├── coding-standards.md
│   ├── git-workflow.md
│   ├── review-policy.md
│   └── security.md
└── my-app/                      ← 個別プロジェクトのルート（ここで claude を起動）
    ├── CLAUDE.md                ← プロジェクト固有のエントリポイント
    ├── docs/
    │   ├── stack.md             ← 技術スタック
    │   ├── architecture.md      ← レイヤ設計
    │   ├── db-schema.md         ← DB スキーマ
    │   ├── screens.md           ← 画面構成
    │   ├── api-spec.md          ← ルーティング・認可
    │   └── seeds.md             ← 初期データ
    └── .claude/
        └── commands/
            ├── scaffold-phase1-skeleton.md
            ├── scaffold-phase2-models.md
            ├── scaffold-phase3-ui.md
            ├── scaffold-phase4-finalize.md
            └── verify.md
```

## 使い方

### 1. 配置

このディレクトリ構造を、新規プロジェクトを作りたい場所にコピーする。

```sh
cp -r rails-claude-md-kit ~/work/my-new-app-parent
cd ~/work/my-new-app-parent/my-app
```

### 2. プロジェクト名・仕様の調整

- `my-app/CLAUDE.md` のプロジェクト名と概要を書き換え
- `my-app/docs/` の各ドキュメントを、実際に作りたい仕様に合わせて編集
  - 特に `db-schema.md`, `screens.md`, `seeds.md` は題材に応じて差し替え

### 3. Claude Code を起動

```sh
cd my-app
claude
```

起動すると、Claude Code は親ディレクトリ方向に `CLAUDE.md` を辿り、チーム共通ルールも自動的に読み込む。

### 4. フェーズ実行

Claude Code のセッションで順番にスラッシュコマンドを実行:

```
/scaffold-phase1-skeleton
```

完了したら:

```
/verify
```

問題なければ次へ:

```
/scaffold-phase2-models
/verify
/scaffold-phase3-ui
/verify
/scaffold-phase4-finalize
/verify
```

各フェーズの間で、Claude Code からの確認や質問に答えながら進める。

### 5. 完成後

`bin/setup` → `bin/dev` で起動。`docs/seeds.md` のテストアカウントでログインして動作確認。

## カスタマイズのコツ

- **別の業務系プロジェクトに転用するとき**: `my-app/docs/` の中身だけ差し替えれば、コマンドや方針はそのまま使える。
- **PHP/Laravel 版を作るとき**: `docs/stack.md` を Laravel 用に、`.claude/commands/` の中身を artisan ベースに書き換える。フェーズ分割の考え方（雛形 → モデル → UI → 仕上げ）は共通。
- **チーム共通ルールの強化**: `team-rules/` 配下にファイルを追加し、ルート `CLAUDE.md` で `@team-rules/xxx.md` でインクルードする。
- **個人設定**: 個人だけのルール（好みのエディタ設定等）は `~/.claude/CLAUDE.md` または各リポジトリの `CLAUDE.local.md`（gitignore 推奨）に置く。

## CLAUDE.md の文法メモ

- `@パス` で他の Markdown を再帰的に読み込める
- 階層: グローバル (`~/.claude/CLAUDE.md`) < 親ディレクトリの CLAUDE.md < 起動ディレクトリの CLAUDE.md < CLAUDE.local.md の順に重ね合わせ
- `.claude/commands/*.md` は `/コマンド名` でスラッシュコマンドとして呼べる（YAML frontmatter で `description` を付けると一覧で表示される）

## 注意

- ローカル PC では Ruby 3.3.x と PostgreSQL 16 が必要。Docker で揃える場合は `docs/stack.md` に Docker セクションを追加して指示してください。
- 生成中に Claude Code が破壊的操作を行おうとした際は確認が入ります。落ち着いて承認・拒否してください。
- 想定外の挙動になったら、その時点で `/verify` をかけて原因を切り分けるのが早道です。
