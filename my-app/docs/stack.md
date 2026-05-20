# 技術スタック

## ランタイム

| 項目 | バージョン | 備考 |
|---|---|---|
| Ruby | 3.3.x | `.ruby-version` で固定 |
| Rails | 8.1.x | フルスタック構成 |
| Node.js | 22.x (Active LTS) | importmap 利用のため最小限 |
| MySQL | 8.x | 開発は Docker (`compose.yaml`)、本番はマネージド |
| Docker | 24.x 以上 | 開発時の DB 起動に必須 |
| Docker Compose | v2 (`docker compose` コマンド) | `docker-compose` (旧 v1) は使わない |

## フレームワーク・主要 Gem

「手動追加」列が ✅ の Gem は `rails new` で自動追加されないため Gemfile に手動追記する。— は `rails new --database=mysql --css=tailwind --javascript=importmap` で自動追加される。

| Gem | 用途 | 手動追加 | Gemfileグループ |
|---|---|---|---|
| `rails` (~> 8.1) | フレームワーク本体 | — | — |
| `mysql2` | MySQL アダプタ | — | — |
| `puma` | アプリサーバ | — | — |
| `importmap-rails` | JS 配信 | — | — |
| `turbo-rails` | Hotwire / Turbo | — | — |
| `stimulus-rails` | Hotwire / Stimulus | — | — |
| `tailwindcss-rails` | CSS | — | — |
| `devise` | 認証 | ✅ | ルート |
| `devise-i18n` | Devise 日本語化 | ✅ | ルート |
| `pundit` | 認可 | ✅ | ルート |
| `kaminari` | ページネーション | ✅ | ルート |
| `ransack` | 検索 | ✅ | ルート |

### 開発・テスト用

| Gem | 用途 | 手動追加 | Gemfileグループ |
|---|---|---|---|
| `rubocop-rails-omakase` | Lint | — | — |
| `brakeman` | セキュリティスキャン | ✅ | `:development, :test` |
| `bundler-audit` | 依存脆弱性チェック | ✅ | `:development, :test` |
| `capybara` | システムテスト | — | — |
| `selenium-webdriver` | ブラウザ駆動 | — | — |
| `factory_bot_rails` | テストデータ | ✅ | `:development, :test` |
| `faker` | ダミーデータ | ✅ | `:development, :test` |
| `letter_opener` | 開発時メール確認 | ✅ | `:development` |
| `simplecov` | テストカバレッジ計測（HTML レポート出力） | ✅ | `:test` |

## ジョブ・キャッシュ・WebSocket

Rails 8 標準の Solid シリーズ（Solid Queue / Solid Cache / Solid Cable）が
`rails new` で自動的に組み込まれるが、**本プロジェクトでは現時点で利用しない**。

- 非同期ジョブ・キャッシュ・リアルタイム通信を必要とする機能は本フェーズでは作らない
- ただし将来の拡張に備えて、`rails new` が生成した Solid 関連のファイル・
  Gem・マイグレーションは**削除せずそのまま残す**
  - `Gemfile` の `solid_queue`, `solid_cache`, `solid_cable`
  - `config/cache.yml`, `config/queue.yml`, `config/cable.yml`
  - 関連マイグレーション（`db:migrate` でそのまま適用する）
- `Rails.cache` は development では `:memory_store`、test では `:null_store` を使う（Rails デフォルト）
- メール送信（Devise のパスワード再発行等）は同期送信で良い。
  development では `letter_opener` で確認するため非同期化は不要
- `Procfile.dev` に Solid Queue のワーカー（`bin/jobs`）は**追加しない**。
  `rails new` が自動で追加した場合は削除する
- Sidekiq / Resque などの代替ジョブキューも導入しない。Redis も追加しない

## ビュー / フロント

- ERB + Hotwire (Turbo Frames / Turbo Streams)
- 軽い動的処理は Stimulus
- CSS は Tailwind ユーティリティ中心
- アイコンは Heroicons の SVG をインライン

## 起動・開発コマンド

| コマンド | 用途 |
|---|---|
| `docker compose up -d db` | 開発用 DB コンテナ起動 |
| `docker compose down` | DB コンテナ停止 |
| `docker compose down -v` | DB を完全初期化（ボリュームごと削除） |
| `bin/setup` | 初回セットアップ（DB 起動 + bundle + db:prepare） |
| `bin/dev` | 開発サーバ起動（rails + tailwind watch） |
| `bin/rails test` | 単体・結合テスト |
| `bin/rails test:system` | システムテスト |
| `bin/rubocop` | Lint |
| `bin/brakeman --no-pager` | セキュリティスキャン |

## Procfile.dev（正規形）

```
web: bin/rails server
css: bin/rails tailwindcss:watch[always]
```

`bin/jobs`（Solid Queue ワーカー）は含めない。
`tailwindcss:watch[always]` は非 TTY 環境での即終了を防ぐために必須。

## ディレクトリ規約

標準 Rails 構成に加えて以下を使う:

- `app/services/` — ビジネスロジックの Service オブジェクト
- `app/policies/` — Pundit ポリシー
- `app/components/` — ViewComponent（必要になったら）

## 環境変数

`.env` で管理。`.env.example` を必ず同期する。

```
DB_USERNAME=app
DB_PASSWORD=app_password
DB_HOST=127.0.0.1
DB_PORT=3306
RAILS_MASTER_KEY=（config/master.key の中身）
DEFAULT_FROM_EMAIL=no-reply@example.local
```

> **注意**: `DATABASE_URL` は設定しない。`database.yml` は `DB_*` の個別変数で接続情報を受け取る設計。`DATABASE_URL` と個別変数を混在させると優先順位が複雑になりデバッグが困難になる。

## 開発 DB（Docker）

開発環境の MySQL はリポジトリ同梱の `compose.yaml` で起動する。

| 操作 | コマンド |
|---|---|
| 起動 | `docker compose up -d db` |
| 停止 | `docker compose down` |
| 完全初期化（データ消去）| `docker compose down -v` |
| ログ確認 | `docker compose logs -f db` |
| MySQL CLI 接続 | `docker compose exec db mysql -uapp -papp_password bookkeeper_development` |
| 疎通確認 | `docker compose exec db mysqladmin ping -uroot -proot_password` |

### 設計方針

- **Rails 本体はホスト側で動かす**。アプリまでコンテナ化はしない（エディタ統合・ファイル同期・パフォーマンスの観点から）。
- **DB のみ Docker で起動する**。各開発者のホスト環境を MySQL のバージョンや設定で汚さないため。
- データは名前付きボリューム `db-data` に永続化される。
- ポートはセキュリティのため `127.0.0.1:3306` にバインドする（外部公開しない）。

### 接続情報

| 項目 | 値 |
|---|---|
| ホスト | `127.0.0.1` |
| ポート | `3306` |
| データベース | `bookkeeper_development` / `bookkeeper_test` |
| アプリ用ユーザ | `app` / `app_password` |
| root ユーザ | `root` / `root_password` |

`bookkeeper_test` は `bin/rails db:create` で自動生成される。

## MySQL 設定の規約

### `config/database.yml`

以下の設定を必ず含める:

```yaml
default: &default
  adapter: mysql2
  encoding: utf8mb4
  collation: utf8mb4_0900_ai_ci
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: <%= ENV.fetch("DB_USERNAME", "app") %>
  password: <%= ENV.fetch("DB_PASSWORD", "app_password") %>
  host: <%= ENV.fetch("DB_HOST", "127.0.0.1") %>
  port: <%= ENV.fetch("DB_PORT", "3306") %>

development:
  <<: *default
  database: bookkeeper_development

test:
  <<: *default
  database: bookkeeper_test

production:
  <<: *default
  database: bookkeeper_production
  username: <%= ENV["BOOKKEEPER_DATABASE_USERNAME"] %>
  password: <%= ENV["BOOKKEEPER_DATABASE_PASSWORD"] %>
  host: <%= ENV["BOOKKEEPER_DATABASE_HOST"] %>
```
