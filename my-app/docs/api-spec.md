# ルーティング仕様

Rails の `config/routes.rb` に展開する想定の宣言的仕様。

## 全体構造

```ruby
Rails.application.routes.draw do
  devise_for :users

  root "home#index"

  authenticate :user do
    resources :books, only: [:index, :show]
    resources :lendings, only: [:index, :show, :create] do
      member do
        patch :return
      end
    end
    resources :notifications, only: [:index] do
      member { patch :read }
    end
    resource :profile, only: [:edit, :update]

    namespace :admin do
      root "dashboard#index"
      resources :users, only: [:index, :show, :update]
      resources :categories
      resources :tags
      resources :books
      resources :lendings, only: [:index, :show] do
        member do
          patch :approve
          patch :reject
        end
      end
      resources :audit_logs, only: [:index]
    end
  end
end
```

## エンドポイント詳細

### `POST /lendings`

- 認証: 要ログイン
- パラメータ: `lending[book_id]`, `lending[note]`
- 成功時: `redirect_to lending_path(@lending), notice: "借用申請を送信しました"`
- 失敗時: 422、書籍詳細にフォーム付きで再描画
- 業務ルール:
  - 在庫 1 以上必須
  - 同一ユーザー × 同一書籍の active な lending（requested / approved / overdue）が既にある場合は不可

### `PATCH /lendings/:id/approve`（admin）

- 認証: 要 admin
- 副作用: state を `approved`、`approved_at` 設定、`due_on = 14 日後`、`books.available_copies -= 1`、通知作成
- すべて 1 トランザクション内（`LendingApprovalService` で実装）

### `PATCH /lendings/:id/return`（メンバー）

- 認証: 要ログイン、本人のみ
- 副作用: state を `returned`、`returned_at` 設定、`books.available_copies += 1`

### `PATCH /lendings/:id/reject`（admin）

- 認証: 要 admin
- 副作用: state を `rejected`、通知作成

## 認可マトリクス

| リソース | member | admin |
|---|---|---|
| 蔵書 read | ✓ | ✓ |
| 蔵書 write | ✗ | ✓ |
| 自分の貸出 read | ✓ | ✓ |
| 他人の貸出 read | ✗ | ✓ |
| 申請承認/却下 | ✗ | ✓ |
| カテゴリ/タグ CRUD | ✗ | ✓ |
| ユーザー管理 | ✗ | ✓ |
| 監査ログ閲覧 | ✗ | ✓ |
