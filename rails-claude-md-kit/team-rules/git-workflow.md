# Git ワークフロー

## ブランチ戦略

- `main`: 常にデプロイ可能な状態。直接コミット禁止。
- `develop`: 統合ブランチ。
- `feature/<issue-number>-<short-desc>`: 機能開発用。
- `fix/<issue-number>-<short-desc>`: バグ修正用。
- `chore/<short-desc>`: ビルド・設定・依存更新等。

## コミットメッセージ

Conventional Commits に準拠する。

```
<type>(<scope>): <subject>

<body>
```

- type: `feat` / `fix` / `docs` / `style` / `refactor` / `test` / `chore`
- subject: 50 文字以内、命令形、末尾ピリオドなし
- body: 必要に応じて「なぜ」を書く

例:
```
feat(books): add stock check before lending

貸出時に在庫が 0 の場合はエラーを返すよう変更。
業務要件 #123 に対応。
```

## プルリクエスト

- タイトルはコミットメッセージと同じ形式。
- 説明には「目的 / 変更点 / 確認方法」を書く。
- レビュー前に必ず CI を通す。
- マージは Squash and merge を基本とする。

## Claude Code が守ること

- ユーザーの明示的な指示なしに `git push` しない。
- `git commit` は単位ごとに分けて行う。1 コミットに無関係な変更を混ぜない。
- `git add .` ではなくファイルを明示的に指定する。
- 既存のコミット履歴を書き換える操作（`rebase -i`, `commit --amend`, `push --force`）は必ず事前確認。
