# セキュリティ

## 秘密情報

- 秘密情報は `.env` または `Rails.application.credentials` で管理する。
- `.env` は `.gitignore` に含め、`.env.example` をリポジトリに含める。
- `config/master.key` は絶対にコミットしない。
- 秘密情報をログ・コンソール出力に含めない。

## 入力と出力

- ユーザー入力はすべて Strong Parameters でホワイトリスト化する。
- SQL は ActiveRecord 経由で書く。生の SQL を書く場合はプレースホルダを使う（文字列結合禁止）。
- HTML 出力は ERB のエスケープ（`<%= %>`）に任せる。`html_safe` / `raw` を使う場合は理由をコメントする。
- ファイルアップロードは MIME / 拡張子 / サイズを必ず検証する。

## 認証・認可

- パスワードは Devise の bcrypt（既定）に任せる。自前で書かない。
- セッション固定化対策（`reset_session` after login）は Devise が処理。
- 管理画面の全アクションは Pundit で認可チェックする。`verify_authorized` を ApplicationController に入れる。
- Mass assignment 防止のため、`permit(:id)` で id を許可しない。

## その他

- `Gemfile.lock` をコミットする。
- `bundle audit` / `bin/brakeman` を CI に組み込む。
- 本番ログに個人情報を出さない（`filter_parameters` を活用）。
