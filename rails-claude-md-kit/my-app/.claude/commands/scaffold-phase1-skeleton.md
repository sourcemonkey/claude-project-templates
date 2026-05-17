---
description: フェーズ1 - Rails 雛形を生成し依存を導入する
---

# Phase 1: スケルトン生成

`docs/stack.md` の技術スタックに従って、Rails アプリの雛形を作成する。

## 実行手順

1. **Ruby バージョン確認**: `ruby -v` で 3.3.x が入っているか確認。なければ `.tool-versions` または `.ruby-version` でユーザーに指示し中断。
2. **PostgreSQL の起動確認**: `pg_isready` で確認。起動していなければユーザーに案内し中断。
3. **Rails アプリ生成**:
   ```
   rails new . \
     --database=postgresql \
     --css=tailwind \
     --javascript=importmap \
     --skip-test=false \
     --skip-jbuilder \
     --skip-keeps \
     --force
   ```
   既存ファイル（CLAUDE.md, docs/, .claude/）は上書きしないよう注意。
4. **Gemfile に追加** (`docs/stack.md` 参照):
   - `devise`
   - `pundit`
   - `kaminari`
   - `ransack`
   - `image_processing`
   - 開発・テスト群: `factory_bot_rails`, `faker`, `letter_opener`, `brakeman`, `bundler-audit`
5. **bundle install**
6. **Devise の初期セットアップ**:
   - `bin/rails generate devise:install`
   - `config/environments/development.rb` に `config.action_mailer.default_url_options = { host: "localhost", port: 3000 }`
   - `letter_opener` を development の delivery_method に設定
7. **Pundit の初期セットアップ**:
   - `bin/rails generate pundit:install`
   - `ApplicationController` に `include Pundit::Authorization` と `after_action :verify_authorized, except: :index, unless: :devise_controller?` を追加
8. **Tailwind の確認**: `bin/rails tailwindcss:install` 済みであることを確認（rails new で実施済みのはず）。
9. **`.env.example` 作成**: `docs/stack.md` の環境変数を記載。
10. **DB 作成**: `bin/rails db:create`
11. **起動確認**: `bin/dev` をバックグラウンドで立ち上げ、`curl -sS -o /dev/null -w "%{http_code}" http://localhost:3000` が 200 を返すことを確認後、サーバを停止。

## このフェーズの完了基準

- [ ] `bin/setup` で初期化できる
- [ ] `bin/dev` で http://localhost:3000 が 200
- [ ] `bin/rails db:create` が成功
- [ ] `Gemfile.lock` がコミット対象に入っている
- [ ] Devise / Pundit の初期化済み

## やらないこと

- モデル生成（Phase 2 で実施）
- Controller / View 生成（Phase 3 で実施）
- Seeds（Phase 4 で実施）

## 完了後

`/verify` を実行してセルフチェックし、結果を「やったこと / 次にやること / 詰まっていること」の 3 点で報告する。
