# 画面構成

## 公開（未ログイン可）

| パス | 画面名 | 概要 |
|---|---|---|
| `GET /` | ランディング | ログイン誘導 |
| `GET /users/sign_in` | ログイン | Devise |
| `GET /users/password/new` | パスワード再発行 | Devise |

## メンバー領域（要ログイン）

| パス | 画面名 | 概要 |
|---|---|---|
| `GET /books` | 蔵書一覧 | 検索 / ページネーション / タグ・カテゴリで絞り込み |
| `GET /books/:id` | 蔵書詳細 | 在庫数表示、借用申請ボタン |
| `POST /lendings` | 借用申請 | 詳細画面から POST |
| `GET /lendings` | 自分の貸出一覧 | 状態フィルタ |
| `GET /lendings/:id` | 自分の貸出詳細 | 返却ボタン |
| `PATCH /lendings/:id/return` | 返却操作 | state を returned に |
| `GET /notifications` | 通知一覧 | 未読/既読切替 |
| `PATCH /notifications/:id/read` | 既読化 | |
| `GET /profile/edit` | プロフィール編集 | |

## 管理者領域（要ログイン + admin ロール）

URL プレフィックス: `/admin/`

| パス | 画面名 | 概要 |
|---|---|---|
| `GET /admin` | ダッシュボード | 申請待ち件数 / 延滞件数 |
| `GET /admin/users` | ユーザー一覧 | 検索 |
| `GET /admin/users/:id` | ユーザー詳細 | 貸出履歴含む |
| `PATCH /admin/users/:id` | ロール変更 | member ⇄ admin |
| `GET /admin/categories` | カテゴリ一覧 | |
| `POST /admin/categories` | カテゴリ作成 | |
| `PATCH /admin/categories/:id` | カテゴリ更新 | |
| `DELETE /admin/categories/:id` | カテゴリ削除 | |
| `GET /admin/tags` | タグ一覧 | |
| `POST /admin/tags` | タグ作成 | |
| `PATCH /admin/tags/:id` | タグ更新 | |
| `DELETE /admin/tags/:id` | タグ削除 | |
| `GET /admin/books` | 蔵書一覧 | |
| `GET /admin/books/new` | 蔵書登録 | |
| `POST /admin/books` | 蔵書作成 | |
| `GET /admin/books/:id/edit` | 蔵書編集 | |
| `PATCH /admin/books/:id` | 蔵書更新 | |
| `DELETE /admin/books/:id` | 蔵書削除 | |
| `GET /admin/lendings` | 貸出申請一覧 | 状態フィルタ |
| `PATCH /admin/lendings/:id/approve` | 申請承認 | |
| `PATCH /admin/lendings/:id/reject` | 申請却下 | |
| `GET /admin/audit_logs` | 監査ログ | 検索 / 日付絞り込み |

## レイアウト

### `application.html.erb`（メンバー向け）

```
┌─────────────────────────────────────────────┐
│ Header: ロゴ | 蔵書 | 自分の貸出 | 通知🔔 | ユーザー名▾ │
├─────────────────────────────────────────────┤
│                                             │
│           {{ yield }}                       │
│                                             │
├─────────────────────────────────────────────┤
│ Footer: © Company                           │
└─────────────────────────────────────────────┘
```

### `admin.html.erb`（管理者向け）

```
┌─────────────────────────────────────────────┐
│ Header: BookKeeper Admin   | ← メンバー画面へ | ▾ │
├──────────┬──────────────────────────────────┤
│ Sidebar  │                                  │
│ - Dash   │       {{ yield }}                │
│ - Users  │                                  │
│ - Books  │                                  │
│ - Cat.   │                                  │
│ - Tags   │                                  │
│ - Lend.  │                                  │
│ - Audit  │                                  │
└──────────┴──────────────────────────────────┘
```

## 画面共通の作法

- 一覧画面はページネーション（Kaminari, 25 件/ページ）。
- 検索フォームは Ransack。
- 削除は確認ダイアログ必須（`data: { turbo_confirm: "..." }`）。
- フラッシュメッセージは画面上部、Tailwind の色でステータス表示（notice = 緑、alert = 赤）。
- フォームエラーは入力欄の直下に赤字で表示。

### ボタン・ラベルの標準（システムテスト記述時の参照用）

| 場面 | ラベル |
|---|---|
| 作成・編集フォームの submit（全リソース共通） | `保存` |
| 削除ボタン | `削除` |
| ユーザーのロール変更 submit | `変更する` |
| ユーザーのロール変更 select の label | `ロール変更` |
| 通知の既読化ボタン | `既読` |
| カテゴリフォームの name フィールド label | `カテゴリ名` |
| タグフォームの name フィールド label | `タグ名` |
