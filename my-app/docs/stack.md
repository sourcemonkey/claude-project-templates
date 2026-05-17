# 技術スタック

## ランタイム

| 項目 | バージョン | 備考 |
|---|---|---|
| Ruby | 3.3.x | `.ruby-version` で固定 |
| Rails | 7.2.x | フルスタック構成 |
| Node.js | 20.x | importmap 利用のため最小限 |
| MySQL | 8.x | 開発は Docker (`compose.yaml`)、本番はマネージド |
| Docker | 24.x 以上 | 開発時の DB 起動に必須 |
| Docker Compose | v2 (`docker compose` コマンド) | `docker-compose` (旧 v1) は使わない |

## フレームワーク・主要 Gem

| Gem | 用途 |
|---|---|
| `rails` (~> 7.2) | フレームワーク本体 |
| `mysql2` | MySQL アダプタ |
| `puma` | アプリサーバ |
| `importmap-rails` | JS 配信 |
| `turbo-rails` | Hotwire / Turbo |
| `stimulus-rails` | Hotwire / Stimulus |
| `tailwindcss-rails` | CSS |
| `devise` | 認証 |
| `pundit` | 認可 |
| `kaminari` | ページネーション |
| `ransack` | 検索 |
| `image_processing` | Active Storage 用 |

### 開発・テスト用

| Gem | 用途 |
|---|---|
| `rubocop-rails-omakase` | Lint |
| `brakeman` | セキュリティスキャン |
| `bundler-audit` | 依存脆弱性チェック |
| `capybara` | システムテスト |
| `selenium-webdriver` | ブラウザ駆動 |
| `factory_bot_rails` | テストデータ |
| `faker` | ダミーデータ |
| `letter_opener` | 開発時メール確認 |

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
| `bin/setup` | 初回セットアップ（DB 起動 + bundle + db:setup） |
| `bin/dev` | 開発サーバ起動（rails + tailwind watch） |
| `bin/rails test` | 単体・結合テスト |
| `bin/rails test:system` | システムテスト |
| `bin/rubocop` | Lint |
| `bin/brakeman --no-pager` | セキュリティスキャン |

## ディレクトリ規約

標準 Rails 構成に加えて以下を使う:

- `app/services/` — ビジネスロジックの Service オブジェクト
- `app/policies/` — Pundit ポリシー
- `app/components/` — ViewComponent（必要になったら）

## 環境変数

`.env` で管理。`.env.example` を必ず同期する。

```
DATABASE_URL=mysql2://app:app_password@127.0.0.1:3306/bookkeeper_development
DB_USERNAME=app
DB_PASSWORD=app_password
DB_HOST=127.0.0.1
DB_PORT=3306
RAILS_MASTER_KEY=（config/master.key の中身）
DEFAULT_FROM_EMAIL=no-reply@example.local
```

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
