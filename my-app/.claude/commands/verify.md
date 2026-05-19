---
description: 現在のフェーズの完了基準を満たしているかセルフチェックする
---

# Verify: セルフチェック

直前に完了したフェーズの完了基準を満たしているか確認する。

## 共通チェック

すべてのフェーズで以下を確認:

1. `bin/rails db:migrate:status` — 全マイグレーションが up
2. `bin/rails test` — 失敗なし（Phase 1 ではテストファイルが少なくても OK）
3. `bin/rubocop` — 違反 0（自動修正可能なものは `bin/rubocop -A`）
4. Git status — 意図しない変更がないか

## フェーズ別チェック

### Phase 1 完了時

- [ ] `Gemfile` に必要 Gem がすべて記載
- [ ] `bin/rails db:create` 成功済み
- [ ] `bin/dev` で 200 が返る
- [ ] `.env.example` 存在

### Phase 2 完了時

- [ ] `db/schema.rb` の各テーブル定義が `docs/db-schema.md` と一致
- [ ] 各モデルに enum / validations / associations が定義済み
- [ ] CHECK 制約が存在（books の available_copies）
- [ ] `bin/rails test:models` all green

### Phase 3 完了時

- [ ] `bin/rails routes` の出力が `docs/api-spec.md` の全エンドポイントを含む
- [ ] 各リソースに Pundit Policy が存在
- [ ] レイアウト `application.html.erb` と `admin.html.erb` が存在
- [ ] Service オブジェクトが `app/services/` に存在
- [ ] 主要画面が（空でも）500 にならない

### Phase 4 完了時

- [ ] `coverage/index.html` が生成され、行カバレッジが 80% 以上
- [ ] `db:reset` 後に `db:seed` が成功
- [ ] seeds 投入後にログインして主要画面が見える
- [ ] `bin/rails test:system` all green
- [ ] `bin/brakeman --no-pager` High 警告 0
- [ ] README.md 完備

## 報告フォーマット

チェック結果を以下の形式で報告:

```
✅ クリア: <項目>
❌ 未達: <項目> — 原因: <推測>、対処案: <提案>
⚠️  確認要: <項目> — ユーザー判断が必要な理由
```

未達がある場合、次のフェーズに進まずに対処する。
