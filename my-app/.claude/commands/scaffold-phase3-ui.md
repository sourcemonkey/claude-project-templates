---
description: フェーズ3 - Controller / View / Policy を生成し UI を完成させる
---

# Phase 3: UI（Controller / View / Policy）

`docs/screens.md` と `docs/api-spec.md` に従って画面と認可を構築する。ルーティング定義・エンドポイント仕様・認可マトリクスはすべて `docs/api-spec.md` が一次情報。画面構成は `docs/screens.md` が一次情報。

## 実行順序

1. **ルーティング**: `docs/api-spec.md` の「全体構造」の通りに `config/routes.rb` を記述
2. **レイアウト**: `application.html.erb`（メンバー用）と `admin.html.erb`（管理者用）を作成。ヘッダ / フッタ / サイドバーの構造は `docs/screens.md` の「レイアウト」セクション参照
3. **ApplicationController**:
   - `include Pundit::Authorization`
   - `rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized`
   - `before_action :authenticate_user!`（root と Devise を除く）
4. **Admin::BaseController**:
   - `ApplicationController` を継承
   - `before_action :require_admin`
   - `layout "admin"`
5. **メンバー領域 Controller / View**:
   - `HomeController#index`
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
7. **Pundit Policy**: リソースごとに `XxxPolicy` を作成。認可ルールは `docs/api-spec.md` の「認可マトリクス」通り
8. **Service オブジェクト**:
   - `LendingRequestService` — 申請（在庫チェック + Lending 作成）
   - `LendingApprovalService` — 承認（state 変更 + 在庫減算 + 通知 + 監査ログ、トランザクション内）
   - `LendingReturnService` — 返却（state 変更 + 在庫増加）
   - `LendingRejectionService` — 却下

各 Service の副作用は `docs/api-spec.md` の「エンドポイント詳細」参照。

## 画面実装の注意

- **検索**: Ransack（`@q = Book.ransack(params[:q])` → `@books = @q.result.page(params[:page])`）
- **ページネーション**: Kaminari の `paginate @books`
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

Controller では各アクションで `authorize @book` を呼ぶ。`index?` 系で絞り込みが必要な場合（自分の貸出のみ表示など）は `Scope` を実装。

## テスト

最低限のシステムテスト（Capybara）を書く。網羅すべき観点:

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
