# Issue 000: Quota Monitoring の IAM モード対応

## Why (背景・動機)

現行の Quota Check API は JWT Authorizer で OIDC トークンを検証するため、SSO 無効（認証モード: None / IAM Identity Center）では Quota スタックのデプロイ自体が失敗する。IAM User や IAM Identity Center で Bedrock を利用するユーザーにもクォータ制御を提供したい。

## What (何を変えるか)

Quota Check API に IAM 認証モード（SigV4）を追加し、caller ARN からユーザーを特定してクォータチェックを行えるようにする。

## Who (対象ユーザー)

- IAM User で直接 Bedrock にアクセスしている開発者
- IAM Identity Center (AWS SSO) 経由で Bedrock にアクセスしている開発者
- 上記ユーザーを管理する IT 管理者

## Where (変更箇所)

- `deployment/infrastructure/quota-monitoring.yaml` — AuthMode パラメータ追加、IAM Authorizer 分岐
- `deployment/infrastructure/quota_check/index.py` — ARN パースによるユーザー特定
- `source/claude_code_with_bedrock/cli/commands/deploy.py` — AuthMode 判定
- `source/claude_code_with_bedrock/cli/commands/init.py` — SSO 無効時のクォータ有効化許可
- `source/otel_helper/__main__.py` — `@anonymous` サフィックス除去 (Issue 004 前提)

## When (優先度・時期)

優先度: High
前提条件: Issue 004 (OTEL メトリクスのユーザー特定) を先に解決する必要あり

## How (解決策)

詳細は以下を参照:
- [requirement.md](./requirement.md) — 機能要件・非機能要件
- [design.md](./design.md) — 詳細設計
