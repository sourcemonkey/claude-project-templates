---
description: フェーズ1 - Rails 雛形を生成し依存を導入する（DB は Docker 上の MySQL）
---

# Phase 1: スケルトン生成

`docs/stack.md` の技術スタックに従って、Rails アプリの雛形を作成する。
DBMS は **MySQL 8.x** を **Docker コンテナ** で起動して利用する。
Rails 本体はホスト側で動かす。

## 実行手順

### 1. 事前確認

1. **Ruby バージョン確認**: `ruby -v` で 3.3.x が入っているか確認。なければ `.tool-versions` または `.ruby-version` でユーザーに指示し中断。
2. **Docker の確認**:
   - `docker version` で Docker Engine が利用可能か確認
   - `docker compose version` で Compose v2 が利用可能か確認
   - どちらかが不可ならユーザーに「Docker Desktop か Docker Engine + Compose v2 をインストールしてください」と案内し中断
3. **ポート 3306 の空き確認**:
   - `lsof -i :3306` または `nc -z 127.0.0.1 3306` で確認
   - 既に使用中ならユーザーに案内し、停止または別ポート利用を判断してもらう

### 2. compose.yaml の作成

プロジェクトルートに `compose.yaml` を作成する。

```yaml
services:
  db:
    image: mysql:8.4
    container_name: bookkeeper-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: bookkeeper_development
      MYSQL_USER: app
      MYSQL_PASSWORD: app_password
    ports:
      - "127.0.0.1:3306:3306"
    volumes:
      - db-data:/var/lib/mysql
      - ./docker/mysql/conf.d:/etc/mysql/conf.d:ro
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_0900_ai_ci
      - --default-authentication-plugin=caching_sha2_password
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-uroot", "-proot_password"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  db-data:
```

ポイント:
- `ports` は `127.0.0.1:3306` にバインド（外部公開しない）
- charset / collation を `docs/stack.md` の規約と一致
- `healthcheck` を入れて起動完了の判定を簡単にする

あわせて `docker/mysql/conf.d/.keep` を作成しておく（追加設定を入れたくなったときの拡張ポイント）。

### 3. DB コンテナの起動

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

### 4. Rails アプリ生成

```sh
rails new . \
  --database=mysql \
  --css=tailwind \
  --javascript=importmap \
  --skip-test=false \
  --skip-jbuilder \
  --skip-keeps \
  --force
```

既存ファイル（`CLAUDE.md`, `docs/`, `.claude/`, `compose.yaml`, `docker/`）は上書きしないよう注意。

### 5. config/database.yml の調整

- `encoding: utf8mb4` を明示
- `collation: utf8mb4_0900_ai_ci` を明示
- `host` / `username` / `password` / `port` を環境変数経由で読み込む形に変更
- Docker の DB は `127.0.0.1:3306` でホスト側に公開されているため、ホスト Rails からはローカル MySQL と同じ接続情報で扱える
- `docs/stack.md` の「MySQL 設定の規約」セクションの YAML サンプルに従う

### 6. Gemfile に追加

`docs/stack.md` 参照:

- `devise`
- `pundit`
- `kaminari`
- `ransack`
- `image_processing`
- 開発・テスト群: `factory_bot_rails`, `faker`, `letter_opener`, `brakeman`, `bundler-audit`
- `mysql2` は `rails new --database=mysql` で既に追加済みのはず、確認のみ

その後 `bundle install`。

### 7. 各種初期化

7-1. **Devise の初期セットアップ**:
   - `bin/rails generate devise:install`
   - `config/environments/development.rb` に `config.action_mailer.default_url_options = { host: "localhost", port: 3000 }`
   - `letter_opener` を development の delivery_method に設定

7-2. **Pundit の初期セットアップ**:
   - `bin/rails generate pundit:install`
   - `ApplicationController` に `include Pundit::Authorization` と `after_action :verify_authorized, except: :index, unless: :devise_controller?` を追加

7-3. **Tailwind の確認**: `bin/rails tailwindcss:install` 済みであることを確認（rails new で実施済みのはず）。

### 8. .env / .env.example の作成

```
DATABASE_URL=mysql2://app:app_password@127.0.0.1:3306/bookkeeper_development
DB_USERNAME=app
DB_PASSWORD=app_password
DB_HOST=127.0.0.1
DB_PORT=3306
RAILS_MASTER_KEY=（config/master.key の中身）
DEFAULT_FROM_EMAIL=no-reply@example.local
```

`.env` は `.gitignore` 済みであることを確認、`.env.example` は同期して commit 対象に含める。

### 9. bin/setup の Docker 対応

Rails が生成した `bin/setup` の冒頭に DB コンテナの起動と待機を組み込む:

```ruby
# bin/setup の DB 関連部分（抜粋）
puts "\n== Starting database container =="
system!("docker compose up -d db")

puts "\n== Waiting for database to be ready =="
30.times do |i|
  break if system("docker compose exec -T db mysqladmin ping -uroot -proot_password > /dev/null 2>&1")
  abort "Database did not become ready in 30 seconds" if i == 29
  sleep 1
end

puts "\n== Preparing database =="
system! "bin/rails db:prepare"
```

### 10. DB の作成と確認

1. `bin/rails db:create`
   - `bookkeeper_development` は compose.yaml 側で既に作成済みだが、`bookkeeper_test` は Rails 側で作る必要がある
   - 失敗する場合の典型原因: コンテナ未起動 / `.env` の認証情報不一致 / `app` ユーザに `bookkeeper_test` への CREATE 権限がない
   - 後者の場合、`app` ユーザにテスト DB 作成権限を付与:
     ```sh
     docker compose exec db mysql -uroot -proot_password -e \
       "GRANT ALL PRIVILEGES ON \`bookkeeper\\_%\`.* TO 'app'@'%'; FLUSH PRIVILEGES;"
     ```
   - エラー時は勝手に `mysql.user` をいじらず、ユーザーに状況を報告して指示を仰ぐ

2. **起動確認**: `bin/dev` をバックグラウンドで立ち上げ、`curl -sS -o /dev/null -w "%{http_code}" http://localhost:3000` が 200 を返すことを確認後、サーバを停止。

## このフェーズの完了基準

- [ ] `compose.yaml` がプロジェクトルートに存在
- [ ] `docker compose up -d db` で DB が起動し、`mysqladmin ping` が成功する
- [ ] `bin/setup` で DB 起動 → セットアップ完了まで一気通貫で動く
- [ ] `bin/dev` で http://localhost:3000 が 200
- [ ] `bin/rails db:create` が成功（development / test 両方）
- [ ] `Gemfile.lock` がコミット対象に入っている
- [ ] Devise / Pundit の初期化済み
- [ ] `.env.example` と `.gitignore` が整備されている

## やらないこと

- モデル生成（Phase 2 で実施）
- Controller / View 生成（Phase 3 で実施）
- Seeds（Phase 4 で実施）
- Rails 本体のコンテナ化（プロジェクト方針として行わない）

## 完了後

`/verify` を実行してセルフチェックし、結果を「やったこと / 次にやること / 詰まっていること」の 3 点で報告する。
