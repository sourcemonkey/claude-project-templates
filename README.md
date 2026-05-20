# Rails プロジェクト生成テンプレートキット

Claude Code に Rails 中規模プロジェクトを 4 フェーズで生成させるための
`CLAUDE.md` / 仕様ドキュメント / スラッシュコマンド一式です。

題材として「蔵書管理 + 貸出記録システム（BookKeeper）」を同梱していますが、
これはあくまで例で、`my-app/docs/` を差し替えれば他の業務系プロジェクトの
雛形としても使えます。

## 構成

```
.
├── .gitignore                   ← テンプレートリポジトリ用の除外設定
├── .ruby-version                ← リポジトリ全体での Ruby バージョン
├── bin/
│   └── init-project.sh          ← 新規リポジトリの初期化スクリプト
├── CLAUDE.md                    ← チーム共通ルール（このリポジトリ配下全体に適用）
├── env.example                  ← 環境変数のサンプル（リポジトリルート用）
├── team-rules/                  ← チーム共通ルールの本体
│   ├── coding-standards.md
│   ├── git-workflow.md
│   ├── review-policy.md
│   └── security.md
└── my-app/                      ← 個別プロジェクトのルート（ここで claude を起動）
    ├── .gitignore               ← Rails アプリ用の除外設定（.env / coverage/ 等を含む）
    ├── .ruby-version            ← my-app 配下での Ruby バージョン
    ├── CLAUDE.md                ← プロジェクト固有のエントリポイント
    ├── compose.yaml             ← 開発用 MySQL コンテナ定義
    ├── docker/                  ← Docker 関連の追加設定
    │   └── mysql/conf.d/
    ├── docs/                    ← 仕様ドキュメント
    │   ├── stack.md             ← 技術スタック
    │   ├── architecture.md      ← レイヤ設計
    │   ├── db-schema.md         ← DB スキーマ
    │   ├── screens.md           ← 画面構成
    │   ├── api-spec.md          ← ルーティング・認可
    │   └── seeds.md             ← 初期データ
    └── .claude/
        ├── settings.json        ← Claude Code の権限設定（共有用）
        └── commands/            ← フェーズ別スラッシュコマンド
            ├── scaffold-phase1-skeleton.md
            ├── scaffold-phase2-models.md
            ├── scaffold-phase3-ui.md
            ├── scaffold-phase4-finalize.md
            └── verify.md
```

## ローカル環境の前提

`my-app/docs/stack.md` の前提に従い、ローカル PC に以下が必要です。

| ツール | バージョン | 用途 |
|---|---|---|
| Ruby | 3.3.x | リポジトリ直下と `my-app/` 配下の両方の `.ruby-version` で固定 |
| Node.js | 22.x (Active LTS) | importmap / Tailwind ビルド用 |
| Docker | 24.x 以上 | 開発用 MySQL の起動 |
| Docker Compose | v2 (`docker compose`) | 同上 |
| Claude Code | 最新版 | フェーズ実行 |

DBMS は MySQL 8.x を **Docker コンテナ** で起動する設計です。
ホスト OS への MySQL インストールは不要です。

### Ruby バージョンを 2 箇所で固定している理由

リポジトリ直下と `my-app/` 配下の両方に `.ruby-version` を置いています。
Claude Code が `rails new` や `gem install rails` などのコマンドを実行する際、
カレントディレクトリが `my-app/` の外側になるケースがあるためです。両方に
同じバージョンを書いておけば、どちらのディレクトリで `rbenv` 等が動いても
同じ Ruby が選ばれます。

Ruby バージョンを上げる場合は **両方を同時に更新**してください。

## 使い方

### 1. 配置

このリポジトリを、新規プロジェクトを作りたい場所にコピーします。

```sh
cp -r <このリポジトリ> ~/work/your-new-project
cd ~/work/your-new-project
```

（または git clone してリネームしても構いません。）

### 2. プロジェクト名・仕様の調整

別プロジェクトに転用する場合は、以下を実際に作りたい題材に合わせて編集します。
BookKeeper をそのまま題材として進める場合はスキップで OK です。

- `my-app/CLAUDE.md` のプロジェクト名と概要
- `my-app/docs/` の各ドキュメント
  - 特に `db-schema.md`, `screens.md`, `seeds.md` は題材に応じて差し替え
- `my-app/compose.yaml` のコンテナ名・DB 名（必要なら）

### 3. Claude Code を起動

```sh
cd my-app
claude
```

起動すると、Claude Code は親ディレクトリ方向に `CLAUDE.md` を辿り、
チーム共通ルール（ルートの `CLAUDE.md` + `team-rules/`）も自動的に
読み込みます。

### 4. フェーズ実行

Claude Code のセッションで順番にスラッシュコマンドを実行します。

```
/scaffold-phase1-skeleton
```

完了したら検証:

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

各フェーズの間で、Claude Code からの確認や質問に答えながら進めます。

### 5. 完成後

`my-app/` 配下に Rails アプリ一式が生成されます。

```sh
cd my-app
bin/setup    # MySQL コンテナ起動 + bundle + db:setup
bin/dev      # 開発サーバ起動
```

具体的な起動 URL・テストアカウントは Phase 4 で生成される
`my-app/README.md` 参照。

### 6. 独立リポジトリへの移行（任意）

Phase 1〜4 が完了したら、`bin/init-project.sh` を使ってテンプレートの
git 履歴を切り離し、独立した開発用リポジトリとして初期化できます。

```sh
bash bin/init-project.sh
```

スクリプトはモードを選べます。

- **モード 1**: このリポジトリ自体を開発用リポジトリとして使う（テンプレートの `.git` を作り直す）
- **モード 2**: 別のディレクトリにコピーしてから、そこを新規リポジトリ化する

どちらのモードでも初回コミットを対話形式で行います。（`.gitignore` はテンプレートに同梱済みのため生成不要）

## カスタマイズのコツ

- **別の業務系プロジェクトに転用するとき**: `my-app/docs/` の中身だけ
  差し替えれば、コマンドや方針はそのまま使えます
- **PHP/Laravel 版を作るとき**: `docs/stack.md` を Laravel 用に、
  `.claude/commands/` の中身を artisan ベースに書き換えます。
  フェーズ分割の考え方（雛形 → モデル → UI → 仕上げ）は共通です
- **DB を PostgreSQL にしたいとき**: `docs/stack.md` の MySQL 関連記述、
  `my-app/compose.yaml` のイメージ、Phase 1 のコマンドファイルの 3 箇所を
  書き換えれば対応できます
- **チーム共通ルールの強化**: `team-rules/` 配下にファイルを追加し、
  ルート `CLAUDE.md` で `@team-rules/xxx.md` でインクルードします
- **個人設定**: 個人だけのルール（好みのエディタ設定等）は
  `~/.claude/CLAUDE.md` または各リポジトリの `CLAUDE.local.md`
  （gitignore 推奨）に置きます

## コマンドファイルの書き方

`my-app/.claude/commands/*.md` を新規追加・編集するときの方針です。
新しいフェーズや補助コマンドを追加する場合、既存のコマンドファイル
（`scaffold-phase1-skeleton.md` など）を雛形にしてください。

### 1. 一次情報は docs/ に集約する

仕様（DB スキーマ、画面構成、ルーティング、認可、seeds 内容など）は
`my-app/docs/` 配下のドキュメントが一次情報です。コマンドファイル側に
同じ内容を書かず、`docs/xxx.md` を参照する形にします。

理由:

- 仕様変更時に 2 箇所メンテになるのを避ける
- コマンドファイルが長くなるとトークン消費が増える
- 役割分担を明確にする（コマンドは「何をするか」、docs は「どう書くか」）

### 2. 同梱ファイルは「確認」しない

リポジトリに既に存在するファイル（`compose.yaml`, `.ruby-version`,
`env.example` など）は、コマンドファイル内で内容を再掲しないでください。
テンプレートに同梱した時点で「正」とみなし、存在を前提に使うだけです。

確認手順を書くと、Claude Code が律儀に `cat` や目視確認を実行して
トークンと承認プロンプトを消費します。冗長確認はテンプレートの
存在意義を否定することにもなります。

### 3. コード例を書くのは「実装ブレ防止」のときだけ

具体的なコード例（Ruby / YAML / Tailwind クラス指定など）は、それが
無いと Claude Code が毎回違う実装を選んでしまう場合のみ残します。例:

- `add_check_constraint` の構文（docs/ にない MySQL 固有の書き方）
- Pundit Policy の最小骨格（API として明示すべき形）
- Tailwind のデザイントークン（クラス名のブレ防止）

逆に、仕様として docs/ に書かれているものは例示しません。

### 4. ファイルの構成

各コマンドファイルは以下の構成を取ります。`## 前提` と `## やらないこと` は
内容がある場合のみ設ける任意セクション。`## 実行手順` はフェーズの性質に応じて
`## 手順` / `## 実行順序` などの名称に変えてよい。

```markdown
---
description: （コマンド一覧で表示される 1 行説明）
---

# Phase N: タイトル

（このフェーズで何をするかの 1-2 段落）

## 前提                 ← 任意。テンプレ同梱ファイル等の前提条件がある場合
## 実行手順             ← 必須。番号付きで順序を明示
## このフェーズの完了基準   ← 必須。チェックリスト形式（- [ ]）
## やらないこと         　　　　　← 任意。他フェーズとの責務分担が不明瞭になりやすい場合
## 完了後               ← 必須。/verify 実行を促す
```

各セクションの役割:

| セクション | 必須/任意 | 役割 |
|---|---|---|
| `## 前提` | 任意 | テンプレ同梱ファイル等の前提条件 |
| `## 実行手順` | 必須 | 番号付きで順序を明示 |
| `## このフェーズの完了基準` | 必須 | チェックリスト形式（`- [ ]`） |
| `## やらないこと` | 任意 | 他フェーズの責務との切り分けが必要な場合 |
| `## 完了後` | 必須 | `/verify` 実行を促す |

### 5. 文体・形式

- 命令形（「〜する」「〜を実行」）。「〜しましょう」「〜してください」は避ける
- 番号付きリストは順序が意味を持つときのみ。それ以外は箇条書き
- 詳細な対応関係は Markdown テーブルで表現
- 冗長な前置きや結論の繰り返しは省く

## CLAUDE.md の文法メモ

- `@パス` で他の Markdown を再帰的に読み込めます（深くしすぎると
  コンテキストを食うので 1〜2 階層が目安）
- 階層: グローバル (`~/.claude/CLAUDE.md`) < 親ディレクトリの CLAUDE.md
  < 起動ディレクトリの CLAUDE.md < CLAUDE.local.md の順に重ね合わせ
- `.claude/commands/*.md` は `/コマンド名` でスラッシュコマンドとして
  呼べます（YAML frontmatter で `description` を付けると一覧表示される）

## Claude Code の権限設定について

このテンプレートには `my-app/.claude/settings.json`（共有用）が
同梱されており、Phase 1〜4 でよく使うコマンドパターンを許可リストに
入れてあります。これにより毎回の承認プロンプトが減ります。

セッション中に「Yes, and don't ask again」を選んだ項目は、個人ローカル用の
`my-app/.claude/settings.local.json`（gitignore 対象）に自動で蓄積されます。
そこから共有してよいパターンを選別して `settings.json` にマージしていくと、
次プロジェクトでさらに承認回数が減らせます。

## トラブルシューティング

### 想定外の挙動になった

その時点で `/verify` をかけて原因を切り分けるのが早道です。
破壊的操作の確認が出た際は、落ち着いて承認・拒否してください。

## 注意

- 生成中に Claude Code が破壊的操作（`rm -rf`, `db:drop` 等）を行おうと
  した際は確認が入ります。`settings.json` の `deny` リストにも
  ガードを置いてあります
- `config/master.key` や `.env` はテンプレート同梱の `my-app/.gitignore` で
  除外済みです。`init-project.sh` 実行時に秘密情報がコミット候補に入っていないことを
  確認してください（スクリプト側でも検出ガードを設けています）
