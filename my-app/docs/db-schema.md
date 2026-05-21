# DB スキーマ

## ER 概略

```
users ──< lendings >── books >── categories
              │            │
              │            └─< book_tags >── tags
              │
              └─< notifications
              
audit_logs (独立、polymorphic)
```

## テーブル定義

### users（Devise）

| カラム | 型 | 制約・既定値 |
|---|---|---|
| id | bigint | PK |
| email | string | NOT NULL, UNIQUE |
| encrypted_password | string | NOT NULL |
| name | string | NOT NULL |
| role | integer | NOT NULL, default: 0（enum: `member: 0`, `admin: 1`） |
| reset_password_token | string | UNIQUE |
| reset_password_sent_at | datetime | |
| remember_created_at | datetime | |
| created_at / updated_at | datetime | |

インデックス: `email`, `reset_password_token`

### categories

| カラム | 型 | 制約・既定値 |
|---|---|---|
| id | bigint | PK |
| name | string | NOT NULL, UNIQUE |
| description | text | |
| created_at / updated_at | datetime | |

### books

| カラム | 型 | 制約・既定値 |
|---|---|---|
| id | bigint | PK |
| category_id | bigint | NOT NULL, FK |
| isbn | string | UNIQUE |
| title | string | NOT NULL |
| author | string | NOT NULL |
| publisher | string | |
| published_on | date | |
| total_copies | integer | NOT NULL, default: 1 |
| available_copies | integer | NOT NULL, default: 1 |
| description | text | |
| published | boolean | NOT NULL, default: false |
| created_at / updated_at | datetime | |

インデックス: `category_id`, `isbn`, `title`

制約: `available_copies >= 0`, `available_copies <= total_copies`（CHECK 制約）

### tags

| カラム | 型 | 制約・既定値 |
|---|---|---|
| id | bigint | PK |
| name | string | NOT NULL, UNIQUE |
| created_at / updated_at | datetime | |

### book_tags（中間テーブル）

| カラム | 型 | 制約・既定値 |
|---|---|---|
| id | bigint | PK |
| book_id | bigint | NOT NULL, FK |
| tag_id | bigint | NOT NULL, FK |
| created_at / updated_at | datetime | |

インデックス: `(book_id, tag_id)` UNIQUE

### lendings（貸出記録）

| カラム | 型 | 制約・既定値 |
|---|---|---|
| id | bigint | PK |
| user_id | bigint | NOT NULL, FK |
| book_id | bigint | NOT NULL, FK |
| state | integer | NOT NULL, default: 0（enum: `requested: 0`, `approved: 1`, `returned: 2`, `rejected: 3`, `overdue: 4`） |
| requested_at | datetime | NOT NULL |
| approved_at | datetime | |
| due_on | date | |
| returned_at | datetime | |
| note | text | |
| created_at / updated_at | datetime | |

インデックス: `user_id`, `book_id`, `state`

#### state 遷移ルール

| 現在の state | 遷移先 | 操作 |
|---|---|---|
| `requested` | `approved` | 管理者承認 |
| `requested` | `rejected` | 管理者却下 |
| `approved` / `overdue` | `returned` | メンバー返却 |

`returned` と `rejected` は終端 state（以降の遷移なし）。`overdue` への変更はバッチ相当の操作（seeds では state を直接指定して良い）。

### notifications

| カラム | 型 | 制約・既定値 |
|---|---|---|
| id | bigint | PK |
| user_id | bigint | NOT NULL, FK |
| kind | integer | NOT NULL, default: 0（enum: `lending_approved`, `lending_rejected`, `return_reminder`） |
| title | string | NOT NULL |
| body | text | |
| read_at | datetime | |
| created_at / updated_at | datetime | |

インデックス: `user_id`, `(user_id, read_at)`

### audit_logs

| カラム | 型 | 制約・既定値 |
|---|---|---|
| id | bigint | PK |
| user_id | bigint | FK（操作者、nullable） |
| target_type | string | NOT NULL |
| target_id | bigint | NOT NULL |
| action | string | NOT NULL（例: `create`, `update`, `destroy`, `approve`） |
| changes_json | json | |
| created_at | datetime | |

> 監査ログは不変レコードのため `updated_at` を持たない。マイグレーションでは `t.timestamps` を使わず `t.datetime :created_at, null: false` と明示すること。

インデックス: `(target_type, target_id)`（`user_id` は `t.references :user` で自動生成されるため `add_index` 不要）

## アソシエーション

テーブル定義とER図から方向を導出する。中間テーブル（`book_tags`）は `has_many :through` で結ぶ。

## バリデーション要点

- `User`: email 必須・形式、name 必須
- `Book`: title / author / category 必須、`total_copies >= 1`、`available_copies >= 0`、ISBN は形式チェック（あれば）
- `Lending`: state 遷移バリデーション（requested → approved → returned 等の妥当性）
- `Tag` / `Category`: name 一意

## Ransack 対応

検索機能を持つ画面で使うモデルに定義する。`has_many` を持つモデルは両方、`belongs_to` のみのモデルは `ransackable_attributes` だけ定義する。

| モデル | ransackable_attributes | ransackable_associations |
|---|---|---|
| Book | title, author, publisher, isbn, description, published, category_id, created_at | category, tags, lendings |
| User | name, email, role, created_at | lendings, notifications |
| Lending | state, requested_at, approved_at, due_on, returned_at, user_id, book_id | — |
| AuditLog | action, target_type, target_id, user_id, created_at | — |

## 削除時の挙動（`dependent` オプション）

| アソシエーション | dependent | 理由 |
|---|---|---|
| `User.has_many :lendings` | `:restrict_with_error` | 貸出履歴があるユーザーは削除不可 |
| `User.has_many :notifications` | `:destroy` | |
| `Book.has_many :lendings` | `:restrict_with_error` | 貸出履歴がある書籍は削除不可 |
| `Book.has_many :book_tags` | `:destroy` | |
| `Category.has_many :books` | `:restrict_with_error` | 書籍が紐づくカテゴリは削除不可 |

## テスト teardown でのレコード削除順序

FK 制約を持つテーブルは依存先を後に削除する（ER 図の矢印の逆順）:

```
AuditLog → Notification → Lending → BookTag → Book → Tag → Category → User
```
