# 技術スタック

## ランタイム

| 項目 | バージョン | 備考 |
|---|---|---|
| Ruby | 3.3.x | `.ruby-version` で固定 |
| Rails | 7.2.x | フルスタック構成 |
| Node.js | 20.x | importmap 利用のため最小限 |
| PostgreSQL | 16.x | 開発・本番とも |

## フレームワーク・主要 Gem

| Gem | 用途 |
|---|---|
| `rails` (~> 7.2) | フレームワーク本体 |
| `pg` | PostgreSQL アダプタ |
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
| `bin/setup` | 初回セットアップ（bundle, db:setup, yarn 等） |
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
DATABASE_URL=postgres://localhost/bookkeeper_development
RAILS_MASTER_KEY=（config/master.key の中身）
DEFAULT_FROM_EMAIL=no-reply@example.local
```
