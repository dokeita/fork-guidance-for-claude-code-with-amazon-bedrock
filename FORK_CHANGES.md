# Fork Changes

このドキュメントは、[upstream](https://github.com/aws-solutions-library-samples/guidance-for-claude-code-with-amazon-bedrock) からの変更点をまとめたものです。

## 1. パッケージ管理の移行: Poetry → uv

| 項目 | Before (upstream) | After (fork) |
|------|-------------------|--------------|
| ビルドバックエンド | `poetry-core` | `hatchling` |
| 依存定義 | `[tool.poetry.dependencies]` | `[project] dependencies` (PEP 621) |
| 開発依存 | `[tool.poetry.group.dev.dependencies]` | `[dependency-groups] dev` (PEP 735) |
| ロックファイル | `poetry.lock` | `uv.lock` (新規追加) |
| 実行方法 | `poetry run ccwb ...` | `uv run ccwb ...` |

**変更ファイル**: `source/pyproject.toml`, `source/uv.lock` (新規)

**動機**: uv は Poetry より高速で、PEP 標準に準拠した依存管理を提供する。

## 2. バグ修正: SSO 無効時の init サマリー表示

**ファイル**: `source/claude_code_with_bedrock/cli/commands/init.py`

`ccwb init` で SSO を無効（認証モード: None）にした場合、サマリー表示で `config["okta"]` に無条件アクセスして `KeyError` が発生していた。

```python
# Before: 無条件アクセス → KeyError
table.add_row("OIDC Provider", config["okta"]["domain"])

# After: 存在チェック付き
if config.get("okta"):
    table.add_row("OIDC Provider", config["okta"]["domain"])
else:
    table.add_row("SSO Authentication", "✗ Disabled (using existing AWS credentials)")
```

## 3. バグ修正: SSO 無効時の deploy 後スタック出力表示

**ファイル**: `source/claude_code_with_bedrock/cli/commands/deploy.py`

`ccwb deploy` 完了後に auth スタックの出力を取得しようとするが、SSO 無効時は auth スタックが存在しないためエラーになっていた。

```python
# Before: 無条件で auth スタック出力を取得
outputs = get_stack_outputs(auth_stack, profile.aws_region)

# After: SSO 有効時のみ取得
sso_enabled = getattr(profile, "sso_enabled", True)
outputs = get_stack_outputs(auth_stack, profile.aws_region) if sso_enabled else {}
```

## 4. ドキュメントの Poetry → uv 更新

全ドキュメントの `poetry run` → `uv run`、`poetry install` → `uv sync`、`Poetry` → `uv` を置換。

**対象ファイル** (16ファイル): README.md, QUICK_START.md, CLI_REFERENCE.md, DEPLOYMENT.md, WINDOWS_BUILD_SYSTEM.md, LOCAL_TESTING.md, COWORK_3P.md, QUOTA_MONITORING.md, distribution/deployment-guide.md, distribution/comparison.md, providers/okta-setup.md, providers/auth0-setup.md, providers/generic-oidc-setup.md, providers/cognito-user-pool-setup.md, providers/microsoft-entra-id-setup.md, source/tests/README.md

## 5. 設計ドキュメント (未実装の改修計画)

以下のドキュメントは、Quota 機能を IAM モード（SSO 無効 / IAM Identity Center）でも動作させるための改修計画です。

| ファイル | 内容 |
|----------|------|
| `issues/000-quota-iam-mode-support/requirement.md` | IAM モード対応の要件定義 |
| `issues/000-quota-iam-mode-support/design.md` | 詳細設計（CFn, Lambda, deploy.py, init.py の変更方針） |
| `qa.md` | デプロイ試行時の Q&A まとめ |
| `issues/` | 課題管理ファイル |

### 改修の概要

現行の Quota Check API は JWT Authorizer で OIDC トークンを検証するため、SSO 無効時は動作しない。改修では SigV4 (IAM 認証) モードを追加し、caller ARN からユーザーを特定する。

---

## 変更の適用方法

```bash
cd source
uv sync          # 依存インストール
uv run ccwb init # 初期設定
```
