---
description: フェーズ3 - Controller / View / Policy を生成し UI を完成させる
---

# Phase 3: UI（Controller / View / Policy）

`docs/screens.md` と `docs/api-spec.md` に従って画面と認可を構築する。

## 実行順序

1. **ルーティング**を `docs/api-spec.md` の通りに `config/routes.rb` に記述
2. **レイアウト**: `application.html.erb`（メンバー用）と `admin.html.erb`（管理者用）を作成
3. **ApplicationController**:
   - `include Pundit::Authorization`
   - `rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized`
   - `before_action :authenticate_user!`（root と Devise を除く）
4. **Admin::BaseController**:
   - `ApplicationController` を継承
   - `before_action :require_admin`
   - `layout "admin"`
5. **メンバー領域 Controller / View**:
   - `HomeController#index`（ランディング）
   - `BooksController` (index, show)
   - `LendingsController` (index, show, create, return)
   - `NotificationsController` (index, read)
   - `ProfilesController` (edit, update)
6. **管理者領域 Controller / View**:
   - `Admin::DashboardController#index`
   - `Admin::UsersController` (index, show, update)
   - `Admin::CategoriesController` (full CRUD)
   - `Admin::TagsController` (full CRUD)
   - `Admin::BooksController` (full CRUD)
   - `Admin::LendingsController` (index, show, approve, reject)
   - `Admin::AuditLogsController` (index)
7. **Pundit Policy** をリソースごとに作成（`docs/api-spec.md` の認可マトリクス通り）
8. **Service オブジェクト**:
   - `LendingRequestService` — 申請（在庫チェック + Lending 作成）
   - `LendingApprovalService` — 承認（state 変更 + 在庫減算 + 通知 + 監査ログ、トランザクション内）
   - `LendingReturnService` — 返却（state 変更 + 在庫増加）
   - `LendingRejectionService` — 却下

## 画面実装の注意

- **検索**: Ransack を使う。`@q = Book.ransack(params[:q])` → `@books = @q.result.page(params[:page])`
- **ページネーション**: Kaminari の `paginate @books`
- **Turbo**: `form_with`（デフォルトで Turbo 対応）。削除リンクは `data: { turbo_method: :delete, turbo_confirm: "削除しますか？" }`
- **フラッシュ**: `application.html.erb` の上部で `notice` / `alert` を Tailwind の色で表示
- **エラー表示**: フォーム部分 partial を作成し、`@record.errors.full_messages` を表示

## レイアウトのスタイル

Tailwind の最小構成:

- メンバー用ヘッダ: 白背景、影、ロゴ + ナビ
- 管理者用サイドバー: ダークグレー背景、リンク一覧
- カード: `bg-white rounded-lg shadow p-6`
- ボタン: primary は `bg-indigo-600 text-white px-4 py-2 rounded hover:bg-indigo-700`

## 既定の認可ルール

```ruby
# 例: BookPolicy
class BookPolicy < ApplicationPolicy
  def index?  = true
  def show?   = true
  def create?  = user.admin?
  def update?  = user.admin?
  def destroy? = user.admin?
end
```

Controller では各アクションで `authorize @book` を呼ぶ。

## テスト

最低限のシステムテスト（Capybara）を書く:
- ログインして蔵書一覧が表示できる
- 蔵書詳細から借用申請ができる
- 管理者が申請を承認できる
- 非 admin が `/admin` にアクセスすると 403 になる

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
