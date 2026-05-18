---
description: フェーズ2 - DB スキーマからモデル・マイグレーションを生成する
---

# Phase 2: モデルとマイグレーション

`docs/db-schema.md` の定義に厳密に従ってマイグレーションとモデルを作成する。テーブル定義・カラム制約・インデックス・enum 値・アソシエーションはすべて `docs/db-schema.md` が一次情報。

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

```sh
bin/rails generate devise User name:string role:integer
```

生成されたマイグレーションを `docs/db-schema.md` の `users` テーブル定義に合わせて補正（`role` の `null: false, default: 0` など）。

User モデルに `docs/db-schema.md` のバリデーション要点とアソシエーションを反映。`role` は enum、`lendings` / `notifications` の `has_many` を追加。

### 残りのモデル

各モデルを `bin/rails generate model ...` で雛形作成し、`docs/db-schema.md` の定義（NOT NULL、UNIQUE、CHECK 制約、インデックス、enum 値）に合わせて手書きで補正する。

ジェネレータの自動生成だけでは制約が足りないので、以下を必ず確認:

- すべての FK は `foreign_key: true`
- 必須カラムは `null: false`
- ユニーク制約と CHECK 制約はマイグレーションに明示
- インデックスは `docs/db-schema.md` の通り

### CHECK 制約の書き方

`books` テーブルの `available_copies` には MySQL の CHECK 制約を 2 つ付ける。マイグレーション内での書き方:

```ruby
add_check_constraint :books, "available_copies >= 0",
  name: "books_available_copies_non_negative"
add_check_constraint :books, "available_copies <= total_copies",
  name: "books_available_lte_total"
```

### Lending の state 遷移

state は `docs/db-schema.md` 定義の enum 値で実装する。state 遷移の妥当性は `state_machines` 等の gem を使わず、Model のメソッド（`approve!`, `reject!`, `return!`）として実装する。各メソッド内で「現在の state が許容される遷移元か」をチェックし、不正なら例外または false を返す。

### アソシエーション

`docs/db-schema.md` の「アソシエーション」セクションの通りに、各モデルに `has_many` / `belongs_to` / `has_many :through` を記述する。

### `dependent` オプション

削除時の挙動は以下の方針:

| アソシエーション | dependent |
|---|---|
| `User.has_many :lendings` | `:restrict_with_error`（貸出履歴があるユーザーは削除不可） |
| `User.has_many :notifications` | `:destroy` |
| `Book.has_many :lendings` | `:restrict_with_error` |
| `Book.has_many :book_tags` | `:destroy` |
| `Category.has_many :books` | `:restrict_with_error` |

### マイグレーション実行

```sh
bin/rails db:migrate
```

エラーが出たら止めて報告。勝手に `db:reset` しない。

`rails new` 同梱の Solid Queue / Solid Cache / Solid Cable 用マイグレーション（`solid_queue_*`, `solid_cache_entries`, `solid_cable_messages` 等）もそのまま順に適用される。**これらには手を加えない**。本プロジェクトではテーブルを使わないが、削除しない。

### モデルテスト

各モデルに最低限のバリデーションテストを書く（`test/models/`）。網羅すべき観点:

- presence
- uniqueness
- enum 定義の確認
- アソシエーションの存在

`bin/rails test:models` で all green を確認。

## このフェーズの完了基準

- [ ] `bin/rails db:migrate:status` で全マイグレーションが up（業務テーブル + Solid 系テーブル）
- [ ] `bin/rails test:models` が all green
- [ ] `db/schema.rb` が `docs/db-schema.md` の定義と一致
- [ ] 各モデルのアソシエーション・バリデーション・enum が定義済み

## やらないこと

- Controller / View（Phase 3 で実施）
- Service オブジェクト（Phase 3 で実施）
- Seeds（Phase 4 で実施）
- Solid Queue / Solid Cache / Solid Cable のマイグレーション・モデル・設定ファイルへの手出し

## 完了後

`/verify` を実行し、結果を報告。
