---
description: フェーズ2 - DB スキーマからモデル・マイグレーションを生成する
---

# Phase 2: モデルとマイグレーション

`docs/db-schema.md` の定義に厳密に従ってマイグレーションとモデルを作成する。

## 実行順序

依存関係を踏まえて以下の順で作成する:

1. `User`（Devise）
2. `Category`
3. `Tag`
4. `Book`
5. `BookTag`
6. `Lending`
7. `Notification`
8. `AuditLog`

## 手順

### Devise User

```
bin/rails generate devise User name:string role:integer
```

生成されたマイグレーションを編集:
- `role` に `null: false, default: 0` を追加
- インデックスは Devise 生成済み

User モデルに追記:
```ruby
enum role: { member: 0, admin: 1 }
validates :name, presence: true
has_many :lendings, dependent: :restrict_with_error
has_many :notifications, dependent: :destroy
```

### 残りのモデル

各モデルについて `bin/rails generate model XXX ...` で生成し、生成されたマイグレーションを `docs/db-schema.md` の定義（NOT NULL、UNIQUE、CHECK 制約、インデックス）に合わせて手書きで補正する。

ジェネレータの自動生成だけでは制約が足りないので、必ず以下を確認:

- すべての FK は `foreign_key: true`
- 必須カラムは `null: false`
- ユニーク制約と CHECK 制約はマイグレーションに明示
- インデックスは `docs/db-schema.md` の通り

### Books の CHECK 制約

```ruby
add_check_constraint :books, "available_copies >= 0", name: "books_available_copies_non_negative"
add_check_constraint :books, "available_copies <= total_copies", name: "books_available_lte_total"
```

### Lending の state 遷移

state は enum で:
```ruby
enum state: { requested: 0, approved: 1, returned: 2, rejected: 3, overdue: 4 }
```

state 遷移の妥当性は `state_machines` 等を使わず、Model のメソッド（`approve!`, `reject!`, `return!`）として実装。各メソッド内で「現在の state が許容される遷移元か」をチェック。

### アソシエーション

`docs/db-schema.md` の「アソシエーション」セクションの通りに、各モデルに `has_many` / `belongs_to` / `has_many :through` を記述。

### `dependent` オプション

- `User.has_many :lendings` → `dependent: :restrict_with_error`（貸出履歴があるユーザーは削除不可）
- `User.has_many :notifications` → `dependent: :destroy`
- `Book.has_many :lendings` → `dependent: :restrict_with_error`
- `Book.has_many :book_tags` → `dependent: :destroy`
- `Category.has_many :books` → `dependent: :restrict_with_error`

### マイグレーション実行

```
bin/rails db:migrate
```

エラーが出たら止めて報告。勝手に `db:reset` しない。

### モデルテスト

各モデルに最低限のバリデーションテストを書く（`test/models/`）。
- presence
- uniqueness
- enum 定義の確認
- アソシエーションの存在

`bin/rails test:models` で all green を確認。

## このフェーズの完了基準

- [ ] `bin/rails db:migrate:status` で全マイグレーションが up
- [ ] `bin/rails test:models` が all green
- [ ] `db/schema.rb` が生成され、`docs/db-schema.md` の定義と一致
- [ ] 各モデルのアソシエーション・バリデーション・enum が定義済み

## やらないこと

- Controller / View（Phase 3 で実施）
- Service オブジェクト（Phase 3 で実施）
- Seeds（Phase 4 で実施）

## 完了後

`/verify` を実行し、結果を報告。
