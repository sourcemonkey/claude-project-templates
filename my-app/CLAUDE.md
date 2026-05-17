# Project: 蔵書管理システム (BookKeeper)

社内向けの書籍蔵書管理 + 貸出記録システム。
一般ユーザーは書籍検索と借用申請、管理者は蔵書・貸出・ユーザーを管理する。

## 仕様ドキュメント

@docs/stack.md  
@docs/architecture.md  
@docs/db-schema.md  
@docs/screens.md  
@docs/api-spec.md  
@docs/seeds.md  

## このプロジェクトでの作業方針

- **仕様優先**: docs/ の記述と実装が食い違ったら docs/ が正。違和感があれば実装前に質問する。
- **動く状態を維持**: 各フェーズの終わりに必ず `bin/rails test` と `bin/dev` で起動確認をする。
- **段階的に作る**: 後述のフェーズ順序を守る。先回りで他フェーズの作業をしない。
- **seeds 必須**: ローカルですぐ触れるよう、各モデルに最低 3 件のサンプルデータを seed に入れる。

## 開発フェーズ（順序厳守）

このプロジェクトは 4 フェーズで構築する。各フェーズは `.claude/commands/` のスラッシュコマンドで実行する。

1. `/scaffold-phase1-skeleton` — Rails 雛形 + 依存導入
2. `/scaffold-phase2-models` — DB スキーマ + Model + マイグレーション
3. `/scaffold-phase3-ui` — 認証 + Controller + View + 認可
4. `/scaffold-phase4-finalize` — Seeds + テスト + 起動確認

各フェーズ完了時、`/verify` で完了基準を満たしているかセルフチェックする。

## 完了の定義（プロジェクト全体）

- [ ] `bin/setup` 一発でセットアップ完了
- [ ] `bin/dev` で起動し、ログインから主要画面遷移まで動作
- [ ] `db:seed` で各画面に表示すべきサンプルデータが入る
- [ ] `bin/rails test` が all green
- [ ] `bin/rubocop` が違反 0
- [ ] README に「起動方法」「テストアカウント」が記載されている

## 重要な制約

- **API モードにしない**。フルスタック Rails（ERB + Hotwire）。
- **JS フレームワーク（React/Vue）を導入しない**。Hotwire（Turbo + Stimulus）で完結させる。
- **Devise を使う**。自前認証を書かない。
- **Pundit を使う**。CanCanCan や自前認可ロジックを書かない。
