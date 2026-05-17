# Project: 蔵書管理アプリ (bookkeeper)

@docs/stack.md
@docs/architecture.md
@docs/db-schema.md
@docs/screens.md
@docs/api-spec.md
@docs/seeds.md

## このプロジェクトでの作業方針

- 新規生成時は `docs/` の仕様を**すべて満たすこと**を最優先。
- `bin/setup` 一発で起動可能な状態にする。
- 各機能の実装後に `bin/rails test` を実行。
- 画面は `docs/screens.md` の URL 設計と一致させる。

## フェーズ実行

新規プロジェクトの生成は以下の順で行う:

1. `/scaffold-phase1-skeleton` — Rails 雛形 + 依存導入 + Docker DB 起動
2. `/scaffold-phase2-models` — DB スキーマ + Model + マイグレーション
3. `/scaffold-phase3-ui` — 認証 + Controller + View + 認可
4. `/scaffold-phase4-finalize` — Seeds + テスト + 起動確認

各フェーズ完了時、`/verify` で完了基準を満たしているかセルフチェックする。

## 完了の定義（プロジェクト全体）

- [ ] `docker compose up -d db` で開発用 DB が起動する
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
- **開発 DB は Docker で起動する**。ホスト側に直接 MySQL をインストールしない前提。
- **Rails 本体はホスト側で動かす**。アプリの `Dockerfile` 化や `bin/dev` の docker compose 化は行わない。
