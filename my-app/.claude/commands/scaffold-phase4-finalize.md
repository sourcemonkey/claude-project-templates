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
- 「延滞」状態は seeds で state を直接 `overdue` に設定して良い
- `bin/rails db:seed` で投入。エラー時は `db:reset` する前に原因を報告
- seeds 投入後、admin ダッシュボードで申請待ち件数・延滞件数が表示されることを確認する

### 2. 主要動線のシステムテスト

#### 2-0. SimpleCov の設定（テストを書く前に必ず実施）

`docs/stack.md` の「SimpleCov 設定（正規形）」の通りに `test/test_helper.rb` を設定する。設定後、`bin/rails test` を一度実行して `coverage/index.html` が生成され、かつカバレッジが 0% でないことを確認してから次のステップへ進む。

#### 2-1. テストシナリオの実装

テストを書く前に、テスト対象の View ファイルを Read してボタン名・フィールド label の実際の文字列を確認すること（ボタン名の標準は `docs/screens.md` の「ボタン・ラベルの標準」参照）。推測で書くと不一致による修正ループが発生する。

テスト実装上の注意:
- **`select` の `from:` には label テキストを渡す**（input の `name` 属性ではない）。例: `select "申請中", from: "状態"` ✅ / `from: "state_eq"` ❌
- **承認済み貸出を使う返却テストでは、`create(:lending, :approved, ...)` の前に `book.update!(available_copies: 1)` で在庫を確保すること**。FactoryBot のトレイトは state のみ設定し `available_copies` は操作しないため、返却後に `available_copies > total_copies` になるケースでエラーになる。

`docs/screens.md` の主要動線を Capybara で網羅。網羅すべきシナリオ:

- メンバー: ログイン → 蔵書検索 → 詳細 → 借用申請 → 自分の貸出一覧で確認
- メンバー: 自分の貸出を返却
- 管理者: ログイン → 申請一覧 → 承認 → 通知が作られ在庫が減ること
- 管理者: 書籍 CRUD（作成 → 編集 → 削除）
- 認可: 非 admin が `/admin` にアクセスすると `root_path` へリダイレクトされる
- 認可: 他人の貸出詳細にアクセスすると `root_path` へリダイレクトされる（Pundit）

### 3. bin/setup の整備

`bin/setup` で以下を一発で実行できるよう整備する:

1. `bundle install`
2. `bin/rails db:prepare`
3. `bin/rails db:seed`

DB コンテナの起動と待機は Phase 1 で既に組み込み済み。

### 4. README.md の作成

`my-app/README.md` を新規作成。含めるべき項目:

- プロジェクト概要（1-2 段落）
- 必要なランタイム（`docs/stack.md` の「ランタイム」表からコピー）
- セットアップ手順（`bin/setup`）
- 起動手順（`bin/dev`）
- テストアカウント表（`docs/seeds.md` の「アカウント」表をコピー）
- 主要 URL（`/`, `/admin`）
- テスト実行コマンド
- 関連ドキュメントへのリンク（`docs/` 配下）

### 5. RuboCop 自動修正

```sh
bin/rubocop -A db/seeds.rb test/
```

### 6. 最終チェック

順番に実行:

1. `bin/rails db:reset` ですべて再構築できることを確認
2. `bin/rails test` が all green
3. `coverage/index.html` を確認し、行カバレッジが 80% 以上
4. `bin/rails test:system` が all green
5. `bin/rubocop` が違反 0
6. `bin/brakeman --no-pager` で High 警告 0
7. `bundle audit update && bundle audit` で既知の脆弱性 0
8. `bin/dev` で起動し、以下を curl で確認:
   - `GET /` → 200 または 302（ログインへ）
   - `GET /users/sign_in` → 200
9. ブラウザでアクセスして以下を目視確認（コマンドだけでは見えない部分の最終確認をユーザーに依頼）:
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
