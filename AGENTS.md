# AGENTS.md

## ルール

### Fork 変更の記録

このリポジトリは [aws-solutions-library-samples/guidance-for-claude-code-with-amazon-bedrock](https://github.com/aws-solutions-library-samples/guidance-for-claude-code-with-amazon-bedrock) の fork です。

upstream からの変更（バグ修正、機能追加、設定変更など）を行った場合は、必ず `FORK_CHANGES.md` に変更内容を追記してください。

### Issue 管理

課題や改修計画は `issues/` ディレクトリにマークダウンファイルとして作成してください。

- ファイル名: `NNN-短い説明.md`（例: `004-otel-user-identification-for-iam-mode.md`）
- 完了した Issue は `issues/closed/` に移動
- 保留中の Issue は `issues/pending/` に移動
- 内容は 5W1H を意識して記載する（Why: 背景・動機、What: 何を変えるか、Who: 対象ユーザー、Where: 変更箇所、When: 優先度・時期、How: 解決策）
- requirement.md や design.md などの関連ドキュメントは `issues/NNN-短い説明/` ディレクトリを作成してその中に配置する

### Git コミットメッセージ

[Conventional Commits](https://www.conventionalcommits.org/ja/v1.0.0/) に従ってください。（詳細: [.docs/conventional-commits.md](.docs/conventional-commits.md)）

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

主な type:
- `feat`: 新機能
- `fix`: バグ修正
- `docs`: ドキュメントのみの変更
- `refactor`: リファクタリング
- `chore`: ビルドや補助ツールの変更

破壊的変更がある場合は `!` を付与するか、フッターに `BREAKING CHANGE:` を記載する。
