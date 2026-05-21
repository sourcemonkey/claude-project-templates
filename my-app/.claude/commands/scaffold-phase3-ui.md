---
description: フェーズ3 - Controller / View / Policy を生成し UI を完成させる
---

# Phase 3: UI（Controller / View / Policy）

`docs/screens.md` と `docs/api-spec.md` に従って画面と認可を構築する。ルーティング定義・エンドポイント仕様・認可マトリクスはすべて `docs/api-spec.md` が一次情報。画面構成は `docs/screens.md` が一次情報。

## 実行順序

1. **ルーティング**: `docs/api-spec.md` の「全体構造」の通りに `config/routes.rb` を記述。`root "home#index"` に対応する `HomeController#index`（公開のランディングページ）もあわせて作成する
2. **レイアウト**: `application.html.erb`（メンバー用）と `admin.html.erb`（管理者用）を作成。ヘッダ / フッタ / サイドバーの構造は `docs/screens.md` の「レイアウト」セクション参照
3. **ApplicationController**:
   - `include Pundit::Authorization`
   - `after_action :verify_authorized, unless: -> { devise_controller? || action_name == "index" }`
   - `rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized`（挙動は `@docs/architecture.md` の「認可エラーの挙動」参照）
   - `before_action :authenticate_user!`。root と Devise コントローラは除外する。`HomeController` で `skip_before_action :authenticate_user!, only: :index` を呼ぶ方式を推奨（ApplicationController 側に除外ロジックを書かない）
4. **Admin::BaseController**:
   - `ApplicationController` を継承
   - `before_action :require_admin`（挙動は `@docs/architecture.md` の「認可エラーの挙動」参照）
   - `layout "admin"`
5. **メンバー領域 Controller / View**: `@docs/screens.md` のメンバー領域画面一覧と `@docs/api-spec.md` のルーティング定義から導出すること
6. **管理者領域 Controller / View**: `@docs/screens.md` の管理者領域画面一覧と `@docs/api-spec.md` のルーティング定義から導出すること
7. **Pundit Policy**: リソースごとに `XxxPolicy` を作成。認可ルールは `@docs/api-spec.md` の「認可マトリクス」通り。実装パターン（シングルトンリソースの `policy_class:` 指定等）は `@docs/architecture.md` の「Policy」セクション参照
8. **Service オブジェクト**: `@docs/architecture.md` の「Service 一覧」参照。各 Service の副作用は `@docs/api-spec.md` の「エンドポイント詳細」参照
9. **カスタムエラーページ**: `public/404.html`, `422.html`, `500.html` を Tailwind スタイルに合わせて作成。`config/application.rb` に `config.exceptions_app = routes` は追加しない（`public/` の静的ファイルで十分）
10. **Devise ビュー生成・日本語化**: `bin/rails generate devise:views` を実行し、生成された `app/views/devise/sessions/new.html.erb` のラベルとボタンを日本語化する（`Eメール` / `パスワード` / `ログイン`）。label とボタンのテキストは `docs/screens.md` の「ボタン・ラベルの標準」参照。**これをしないとシステムテストのログインが失敗する**（デフォルトのビューは英語表記のため）

## 画面実装の注意

- **検索**: Ransack（`@q = Book.ransack(params[:q])` → `@books = @q.result.page(params[:page])`）
  - `ransackable_attributes` / `ransackable_associations` は Phase 2 で定義済みのはず。未定義なら `docs/db-schema.md` の「Ransack 対応」セクションを参照して追加する
- **ページネーション**: Kaminari の `paginate @books`。件数は `docs/screens.md` の通り 25件/ページ。`@q.result.page(params[:page]).per(25)` のように `.per(25)` を付ける
- **借用申請フォーム**: `form_with scope: :lending, url: lendings_path` と書く。`scope:` を省くとパラメータが `book_id` でフラットに送信され `params[:lending]` が nil になる
- **Turbo**: `form_with` を使う。削除リンクは `data: { turbo_method: :delete, turbo_confirm: "削除しますか？" }`
- **フラッシュ**: `application.html.erb` の上部で `notice` / `alert` を Tailwind の色で表示
- **エラー表示**: フォーム部分 partial を作成し、`@record.errors.full_messages` を表示

## レイアウトのスタイル（実装ブレ防止のため明示）

Tailwind の最小構成。Claude Code がクラスを毎回違う組み合わせで選ばないよう、以下に揃える:

- メンバー用ヘッダ: 白背景、影、ロゴ + ナビ
- 管理者用サイドバー: ダークグレー背景、リンク一覧
- カード: `bg-white rounded-lg shadow p-6`
- プライマリボタン: `bg-indigo-600 text-white px-4 py-2 rounded hover:bg-indigo-700`
- 危険ボタン（削除等）: `bg-red-600 text-white px-4 py-2 rounded hover:bg-red-700`
- フラッシュ notice: `bg-green-50 text-green-800 border border-green-200 rounded p-3`
- フラッシュ alert: `bg-red-50 text-red-800 border border-red-200 rounded p-3`

## Pundit Policy の書き方（実装ブレ防止のため例示）

リソース全体に共通する形式:

```ruby
class BookPolicy < ApplicationPolicy
  def index?   = true
  def show?    = true
  def create?  = user.admin?
  def update?  = user.admin?
  def destroy? = user.admin?
end
```

Controller では各アクションで `authorize @record` を呼ぶ。`index` アクションは `verify_authorized` から除外されているため `authorize` は不要。絞り込みが必要なリソース（例: Lending は自分の貸出のみ）は `policy_scope` を使う:

```ruby
# 全件表示（BooksController#index 等）
@books = Book.page(params[:page])

# 絞り込みあり（LendingsController#index 等）
@lendings = policy_scope(Lending).page(params[:page])
```

## RuboCop 自動修正

Service・Policy・Controller・View の実装が完了したら自動修正可能な違反を解消する:

```sh
bin/rubocop -A app/controllers/ app/views/ app/policies/ app/services/ test/
```

## テスト

### `application_system_test_case.rb` の設定（テストを書く前に必ず実施）

`rails new` が生成するデフォルトの `test/application_system_test_case.rb` は
`use_transactional_tests` の設定がない（= デフォルト `true`）。この状態では
FactoryBot で作ったデータがテストトランザクション内にしか存在せず、
Selenium ブラウザ（別スレッド）から見えないため**全システムテストが失敗する**。

以下の内容で上書きすること:

```ruby
require "test_helper"

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  include FactoryBot::Syntax::Methods

  driven_by :selenium, using: :headless_chrome, screen_size: [ 1400, 1400 ]

  self.use_transactional_tests = false

  teardown do
    Capybara.reset_sessions!  # テスト間でセッションを確実にリセット（ないとセッション汚染で後続テストが失敗する）
    # FK依存の逆順（順序の根拠は @docs/db-schema.md の「teardown削除順序」セクション参照）
    [ AuditLog, Notification, Lending, BookTag, Book, Tag, Category, User ].each(&:delete_all)
  end
end
```

### `sign_in_as` ヘルパー

Capybara でログイン後の画面操作を行う際、リダイレクト完了を待たずに次の操作を行うと
断続的に失敗するテストになる。各テストファイルの `private` に以下を必ず含めること:

```ruby
def sign_in_as(user)
  visit new_user_session_path
  fill_in "Eメール", with: user.email
  fill_in "パスワード", with: "password123"
  click_button "ログイン"  # click_on ではなく click_button を使う（リンクと区別するため）
  assert_no_current_path new_user_session_path, wait: 5  # リダイレクト完了を待つ
end
```

`assert_no_current_path ... wait: 5` がないと、ログイン画面からの遷移前に後続の操作が
走り、ランダムに失敗するテストになる。フィールド label（`Eメール` / `パスワード` / ボタン `ログイン`）は
`docs/screens.md` の「ボタン・ラベルの標準」参照。

### テストシナリオ

最低限のシステムテスト（Capybara）を書く。網羅すべき観点:

- ログインして蔵書一覧が表示できる
- 蔵書詳細から借用申請ができる
- 管理者が申請を承認できる
- 非 admin が `/admin` にアクセスすると `root_path` へリダイレクトされる（flash[:alert] が表示される）

`bin/rails test:system` で確認。

## このフェーズの完了基準

- [ ] `bin/rails routes` で `docs/api-spec.md` の全ルートが存在
- [ ] 各画面が（データなしでも）500 にならずに表示できる
- [ ] Pundit Policy が全リソースに存在
- [ ] `bin/rails test` が all green
- [ ] `bin/rubocop` が違反 0

## やらないこと

- Seeds の投入（Phase 4 で実施）
- 本番デプロイ設定

## 完了後

`/verify` を実行し、結果を報告。
