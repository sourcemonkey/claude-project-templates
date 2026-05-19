---
description: フェーズ4 - Seeds、テスト、README、起動確認で完成させる
---

# Phase 4: 仕上げ（Seeds + テスト + 起動確認）

最終フェーズ。「最初から動くもの」を完成させる。投入データ・テストアカウント等の具体値は `docs/seeds.md` が一次情報。

## 実行手順

### 1. Seeds の実装

`docs/seeds.md` に記載されたデータ（アカウント、カテゴリ、タグ、書籍、貸出、通知、監査ログ）をそのまま `db/seeds.rb` に実装する。

実装上の注意:

- すべて `find_or_create_by!` で冪等にする（複数回実行しても重複しない）
- 貸出（Lending）を作成する際、書籍の `available_copies` を整合的に更新する
- 「延滞」状態を作る場合、簡単のため state を `overdue` で直接作成して良い
- `bin/rails db:seed` で投入。エラー時は `db:reset` する前に原因を報告

### 2. 主要動線のシステムテスト

#### 2-0. SimpleCov の設定（テストを書く前に必ず実施）

`test/test_helper.rb` の**冒頭（他の require より前）**に以下を追記する。記述位置を間違えると計測漏れが発生するため厳守。

```ruby
require "simplecov"
SimpleCov.start "rails" do
  add_filter "/test/"
  add_filter "/config/"
  add_filter "/vendor/"
  add_group "Services", "app/services"
  add_group "Policies", "app/policies"
end
```

`.gitignore` に `coverage/` を追記する。

追記後、`bin/rails test` を一度実行して `coverage/index.html` が生成されることを確認してから次のステップへ進む。

#### 2-1. `application_system_test_case.rb` の設定（テストを書く前に必ず実施）

`rails new` が生成するデフォルトの `test/application_system_test_case.rb` は
`use_transactional_tests` の設定がない（= デフォルト `true`）。この状態では
FactoryBot で作ったデータがテストトランザクション内にしか存在せず、
Selenium ブラウザ（別スレッド）から見えないため**全システムテストが失敗する**。

テストを 1 行も書く前に、以下の内容で上書きすること:

```ruby
require "test_helper"

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  include FactoryBot::Syntax::Methods

  driven_by :selenium, using: :headless_chrome, screen_size: [ 1400, 1400 ]

  self.use_transactional_tests = false

  teardown do
    # FK依存の逆順（順序の根拠は @docs/db-schema.md の「teardown削除順序」セクション参照）
    [ AuditLog, Notification, Lending, BookTag, Book, Tag, Category, User ].each(&:delete_all)
  end
end
```

設定後、`bin/rails test:system` を空のテストファイルで一度実行してエラーなく起動することを確認してから、各テストを実装する。

#### 2-1-1. `sign_in_as` ヘルパーのテンプレート

Capybara でログイン後の画面操作を行う際、リダイレクト完了を待たずに次の操作を行うと
断続的に失敗するテストになる。各テストファイルの `private` に以下を必ず含めること:

```ruby
def sign_in_as(user)
  visit new_user_session_path
  fill_in "メールアドレス", with: user.email
  fill_in "パスワード", with: "password123"
  click_on "ログイン"
  assert_no_current_path new_user_session_path, wait: 5  # リダイレクト完了を待つ
end
```

`assert_no_current_path ... wait: 5` がないと、ログイン画面からの遷移前に後続の操作が
走り、ランダムに失敗するテストになる。

#### 2-2. テストシナリオの実装

`docs/screens.md` の主要動線を Capybara で網羅。網羅すべきシナリオ:

- メンバー: ログイン → 蔵書検索 → 詳細 → 借用申請 → 自分の貸出一覧で確認
- メンバー: 自分の貸出を返却
- 管理者: ログイン → 申請一覧 → 承認 → 通知が作られ在庫が減ること
- 管理者: 書籍 CRUD（作成 → 編集 → 削除）
- 認可: 非 admin が `/admin` にアクセスして 403 になる
- 認可: 他人の貸出詳細にアクセスして 404 / 403 になる

### 3. ダッシュボードの実データ表示

`docs/screens.md` の admin ダッシュボードに、seeds で投入されたデータが表示されることを確認:

- 申請待ち件数
- 延滞件数

### 4. bin/setup の整備

`bin/setup` を以下が一発で動くように整備:

1. `bundle install`
2. `yarn install`（必要なら）
3. `bin/rails db:prepare`
4. `bin/rails db:seed`

DB コンテナの起動と待機は Phase 1 で既に組み込み済み。

### 5. README.md の作成

`my-app/README.md` を新規作成。含めるべき項目:

- プロジェクト概要（1-2 段落）
- 必要なランタイム（`docs/stack.md` の「ランタイム」表からコピー）
- セットアップ手順（`bin/setup`）
- 起動手順（`bin/dev`）
- テストアカウント表（`docs/seeds.md` の「アカウント」表をコピー）
- 主要 URL（`/`, `/admin`）
- テスト実行コマンド
- 関連ドキュメントへのリンク（`docs/` 配下）

### 6. 最終チェック

順番に実行:

1. `bin/rails db:reset` ですべて再構築できることを確認
2. `bin/rails test` が all green
3. `coverage/index.html` を確認し、行カバレッジが 80% 以上
4. `bin/rails test:system` が all green
5. `bin/rubocop` が違反 0
6. `bin/brakeman --no-pager` で High 警告 0
7. `bin/dev` で起動し、以下を curl で確認:
   - `GET /` → 200 または 302（ログインへ）
   - `GET /users/sign_in` → 200
8. ブラウザでアクセスして以下を目視確認（コマンドだけでは見えない部分の最終確認をユーザーに依頼）:
   - 管理者でログインしてダッシュボード
   - メンバーでログインして借用申請

## このフェーズの完了基準（= プロジェクト全体の完成）

- [ ] `bin/setup` 一発でセットアップ完了
- [ ] `bin/dev` で起動して全機能が動作
- [ ] seeds で各画面に表示すべきデータが入る
- [ ] `bin/rails test` および `test:system` が all green
- [ ] カバレッジが 80% 以上（`coverage/index.html` で確認）
- [ ] `bin/rubocop` 違反 0、`bin/brakeman` High 0
- [ ] README にテストアカウント・起動方法が記載

## 完了後

ユーザーに以下を報告:

- できあがった機能の一覧
- テストアカウントと URL
- 既知の制限・未実装事項（あれば）
- 次のステップ提案（CI 設定、Docker 化、本番デプロイ等）
