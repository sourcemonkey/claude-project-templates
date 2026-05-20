---
description: フェーズ1 - Rails 雛形を生成し依存を導入する（DB は Docker 上の MySQL）
---

# Phase 1: スケルトン生成

`docs/stack.md` の技術スタックに従って、Rails アプリの雛形を作成する。
DBMS は **MySQL 8.x** を **Docker コンテナ** で起動して利用する。
Rails 本体はホスト側で動かす。

## 前提

以下はテンプレートに同梱済み。Phase 1 で新規作成しない:

- `my-app/compose.yaml`
- `my-app/docker/mysql/conf.d/.keep`
- `my-app/.ruby-version`
- `my-app/.gitignore`（`.env` 除外・カバレッジ除外など含む）
- ルート直下の `.ruby-version`, `env.example`, `.gitignore`

これらの内容を確認・編集する必要はない（中身は `docs/stack.md` の規約に合致した状態でコミット済み）。

## 実行手順

### 1. 事前確認

1. **Ruby バージョン確認**: `my-app/` 内で `ruby -v` を実行し 3.3.x が出るか確認。出ない場合は `rbenv install` 等でインストールを促し中断。
2. **Docker の確認**:
   - `docker version` で Docker Engine が利用可能か確認
   - `docker compose version` で Compose v2 が利用可能か確認
   - どちらかが不可ならユーザーに「Docker Desktop か Docker Engine + Compose v2 をインストールしてください」と案内し中断
3. **ポート 3306 の空き確認**:
   - `lsof -i :3306` または `nc -z 127.0.0.1 3306` で確認
   - 既に使用中ならユーザーに案内し、停止または別ポート利用を判断してもらう

### 2. DB コンテナの起動

`my-app/` ディレクトリで以下を実行:

1. `docker compose up -d db` で DB コンテナを起動
2. DB が ready になるまで待機:
   ```sh
   for i in $(seq 1 30); do
     if docker compose exec -T db mysqladmin ping -uroot -proot_password > /dev/null 2>&1; then
       echo "MySQL is ready"
       break
     fi
     sleep 1
   done
   ```
3. `docker compose exec db mysql -uapp -papp_password -e "SELECT 1"` で `app` ユーザでログインできることを確認

起動失敗時の典型原因:
- ポート競合（`docker compose logs db` で確認）
- ボリュームに前回データが残っていてユーザ作成がスキップされた（`docker compose down -v` で初期化）

### 3. Rails アプリ生成

`my-app/` 内で実行:

```sh
rails new . \
  --database=mysql \
  --css=tailwind \
  --javascript=importmap \
  --no-skip-test \
  --skip-jbuilder \
  --skip-keeps \
  --skip-git \
  --force
```

`--force` により Rails が生成するファイルは上書きされる。`CLAUDE.md`, `docs/`, `.claude/`, `compose.yaml`, `docker/` は Rails が生成しないため上書きされない。`.ruby-version` は Rails も生成するが、Step 1 で Ruby 3.3.x を確認済みであれば内容は同一になる。

`--skip-git` を指定することで `git init` と `.gitignore` の生成をスキップする。テンプレート同梱の `.gitignore` をそのまま使うためこのフラグが必要。

Rails 8.1 では Solid Queue / Solid Cache / Solid Cable がデフォルト組み込み。本プロジェクトでは使わない方針だが、生成された Gem / 設定ファイル / マイグレーションは**削除せずそのまま残す**（詳細は `docs/stack.md` の「ジョブ・キャッシュ・WebSocket」セクション）。

例外として、`Procfile.dev` に `bin/jobs`（Solid Queue ワーカー）が含まれていたら削除する。

### 4. config/database.yml の調整

`docs/stack.md` の「MySQL 設定の規約」セクションに記載の YAML サンプル通りに修正する。要点:

- `encoding: utf8mb4` / `collation: utf8mb4_0900_ai_ci` を明示
- `host` / `username` / `password` / `port` を環境変数経由で読み込む

Rails 8 が `cache` / `queue` / `cable` 用に別 DB（SQLite など）を定義している場合は、すべて同一の MySQL を使う構成に統一する。

### 5. Gemfile への追加

`docs/stack.md` の「手動追加」列が ✅ の Gem を Gemfile に追記し、`bundle install` を実行する。

### 6. 各種初期化

- **Devise**: `bin/rails generate devise:install`。`config/environments/development.rb` に `config.action_mailer.default_url_options = { host: "localhost", port: 3000 }` を追記。`letter_opener` を development の delivery_method に設定。メール送信は同期送信（`deliver_now`）で良い。
- **devise-i18n**: `config/application.rb` の `Application` クラス内に `config.i18n.default_locale = :ja` を追記する。これにより Devise のビュー・バリデーションメッセージが日本語化される。
- **Pundit**: `bin/rails generate pundit:install`。ApplicationController への設定は Phase 3 でまとめて行う。
- **Tailwind**: `rails new --css=tailwind` で導入済みのはず。確認のみ。

### 7. .env の準備

ルート同梱の `env.example` を `my-app/.env` と `my-app/.env.example` にコピーする:

```sh
cp ../env.example .env
cp ../env.example .env.example
```

`RAILS_MASTER_KEY` の値は `config/master.key` から読み取って `.env` に書き込む（`.env.example` は `RAILS_MASTER_KEY=` のまま空にしておく）。`.env` はテンプレート同梱の `.gitignore` で除外済み。`.env.example` は git 管理対象に含める。

> **注意: `DATABASE_URL` を `.env` に追加しないこと**
>
> このプロジェクトの `database.yml` は `DB_HOST` / `DB_USERNAME` / `DB_PASSWORD` / `DB_PORT` の個別変数で接続情報を受け取る設計になっている。
> `DATABASE_URL` 方式と個別変数方式を混在させると接続設定の優先順位が複雑になりデバッグが困難になる。
> プロジェクト全体で個別変数方式に統一するため、`DATABASE_URL` は設定しないこと。

### 8. bin/setup の Docker 対応

`rails new` が生成した `bin/setup` に、DB コンテナ起動と ready 待機を組み込む。最低限以下の処理が `bin/rails db:prepare` の前に来るようにする:

```ruby
system!("docker compose up -d db")

30.times do |i|
  break if system("docker compose exec -T db mysqladmin ping -uroot -proot_password > /dev/null 2>&1")
  abort "Database did not become ready in 30 seconds" if i == 29
  sleep 1
end

system! "bin/rails db:prepare"
```

### 9. DB の作成と起動確認

1. `bin/rails db:create`
   - DB 名は `docs/stack.md` の「接続情報」表参照。development は compose.yaml 側で作成済み、test はここで作る
   - 失敗時の典型原因: コンテナ未起動 / `.env` の認証情報不一致 / `app` ユーザに test DB 作成権限がない
   - 権限不足の場合、root ユーザで以下を実行（`<prefix>` は `docs/stack.md` の database 名から導出）:
     ```sh
     docker compose exec db mysql -uroot -proot_password -e \
       "GRANT ALL PRIVILEGES ON \`<prefix>\\_%\`.* TO 'app'@'%'; FLUSH PRIVILEGES;"
     ```
   - エラー時は勝手に `mysql.user` をいじらず、状況を報告して指示を仰ぐ
2. **起動確認**: `bin/dev` をバックグラウンドで立ち上げ、`curl -sS -o /dev/null -w "%{http_code}" http://localhost:3000` が 200 を返すことを確認後、サーバを停止（`kill %1` または `pkill -f puma`）。

## このフェーズの完了基準

- [ ] `docker compose up -d db` で DB が起動し、`mysqladmin ping` が成功する
- [ ] `bin/setup` で DB 起動 → セットアップ完了まで一気通貫で動く
- [ ] `bin/dev` で http://localhost:3000 が 200
- [ ] `bin/rails db:create` が成功（development / test 両方）
- [ ] `Gemfile.lock` がコミット対象に入っている
- [ ] Devise / Pundit の初期化済み
- [ ] `my-app/.env` が存在し、`.gitignore` で除外されている
- [ ] `my-app/.env.example` が存在し、コミット対象に含まれている
- [ ] `Procfile.dev` に Solid Queue ワーカー（`bin/jobs`）の行が**含まれていない**

## やらないこと

- モデル生成（Phase 2 で実施）
- Controller / View 生成（Phase 3 で実施）
- Seeds（Phase 4 で実施）
- Rails 本体のコンテナ化（プロジェクト方針として行わない）
- Solid Queue ワーカーの起動設定（本プロジェクトでは非同期ジョブを使わない）
- Sidekiq / Redis の追加
- 同梱ファイル（`compose.yaml`, `.ruby-version`, `env.example`）の編集

## 完了後

`/verify` を実行してセルフチェックし、結果を「やったこと / 次にやること / 詰まっていること」の 3 点で報告する。
