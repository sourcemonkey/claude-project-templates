# コーディング規約

## 共通

- インデントは半角スペース 2 つ。タブ禁止。
- ファイル末尾に改行を入れる。
- 行末の余分な空白を残さない。
- マジックナンバーは定数化する。
- コメントは「なぜ」を書く。「何を」しているかはコード自身に語らせる。

## Ruby / Rails

- RuboCop は `rubocop-rails-omakase` の標準設定に従う。
- メソッド名・変数名は `snake_case`、クラス名は `CamelCase`。
- 真偽値を返すメソッドは `?` で終える（例: `published?`）。
- 早期 return を使ってネストを浅く保つ。
- N+1 を避ける。一覧画面では `includes` / `preload` を必ず検討する。
- Fat Controller を避け、ビジネスロジックは Model か Service オブジェクトに置く。
- View には `if` の連鎖を書かない。Helper か ViewComponent で吸収する。
- `find_by` で nil を返す可能性のあるものは `nil` チェックを必ず行う。
- マイグレーションは可逆にする（`change` で書けないものは `up` / `down` を両方書く）。

## CSS / フロント

- Tailwind のユーティリティを基本とし、独自 CSS は最小限。
- 共通化が必要になったら ViewComponent か partial に切り出す。

## 命名

- リソース名は複数形 (`products`, `users`)。
- boolean カラムは `is_` を付けず、形容詞・過去分詞で（`published`, `archived`）。
- 日時カラムは `_at` 接尾辞、日付は `_on` 接尾辞。
