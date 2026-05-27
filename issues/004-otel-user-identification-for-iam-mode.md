# Issue 004: OTEL メトリクスのユーザー特定依存の解消

## 概要

Quota Monitor Lambda はユーザー使用量を OTEL メトリクスの `user.email` ラベルに依存して取得している。IAM モードでは OTEL Helper がユーザーを正しく識別できない場合があり、クォータ追跡が機能しない可能性がある。

## 背景（Why）

- Quota Monitor Lambda は PromQL クエリ `sum by ("user.email")(increase(claude_code.token.usage[900s]))` でユーザー別使用量を取得
- `user.email` ラベルは OTEL Helper が JWT トークンから抽出して付与している
- IAM モードでは JWT トークンが存在しないため、OTEL Helper が `user.email` を正しく設定できない可能性がある
- OTEL Helper が IAM caller identity からユーザー名を取得するロジックが不完全な場合、メトリクスが匿名化され、クォータ追跡が破綻する

## 対象（Who / What）

- **影響を受けるユーザー**: IAM モードで Quota Monitoring を利用する全組織
- **対象コンポーネント**: OTEL Helper、Quota Monitor Lambda、メトリクスパイプライン

## 現状の動作

```
JWT モード:
  OTEL Helper → JWT decode → email claim → user.email ラベル付与 → CloudWatch

IAM モード:
  OTEL Helper → ??? → user.email ラベル = 不明 or ハッシュ値 → CloudWatch
```

## 解決策の候補（How）

### 案 A: OTEL Helper に STS GetCallerIdentity 呼び出しを追加

OTEL Helper 起動時に `sts:GetCallerIdentity` を呼び、ARN からユーザー名を抽出して `user.email` ラベルに設定する。

```python
# IAM User: arn:aws:iam::123456789012:user/john.doe → "john.doe"
# SSO User: arn:aws:sts::*:assumed-role/AWSReservedSSO_*/john@co.com → "john@co.com"
```

- **メリット**: 追加インフラ不要、既存パイプラインをそのまま活用
- **デメリット**: STS 呼び出しのレイテンシ（初回のみ）、ARN フォーマットへの依存

### 案 B: 環境変数でユーザー名を明示指定

`CLAUDE_CODE_USER` 環境変数を設定し、OTEL Helper がそれを `user.email` に使う。

```bash
export CLAUDE_CODE_USER=$(aws sts get-caller-identity --query Arn --output text | awk -F/ '{print $NF}')
```

- **メリット**: シンプル、確実
- **デメリット**: ユーザーに設定を求める必要がある（install スクリプトで自動化可能）

### 案 C: Quota Monitor Lambda 側で Bedrock CloudTrail ログから使用量を取得

OTEL メトリクスに依存せず、CloudTrail の Bedrock API コールログからユーザー別使用量を集計する。

- **メリット**: OTEL Helper に依存しない、IAM の正確な identity が使える
- **デメリット**: CloudTrail ログの遅延（最大15分）、トークン数の取得が困難（CloudTrail にはトークン数が記録されない）

## 実装場所（Where）

- `source/otel_helper/__main__.py`（ユーザー特定ロジック追加）
- `source/otel_helper/collector-config.yaml`（ラベル設定）
- テスト: IAM User / IAM Identity Center / AssumeRole の各パターン

## 優先度・時期（When）

- **優先度**: High（IAM モード対応の前提条件）
- **前提**: IAM モード対応と同時に実施
- **想定工数**: 2-3日

## 受け入れ条件

- [ ] IAM User で認証した場合、OTEL メトリクスに正しい `user.email` ラベルが付与される
- [ ] IAM Identity Center で認証した場合、メールアドレスが `user.email` に設定される
- [ ] AssumeRole の場合、`role/session` 形式で識別可能
- [ ] Quota Monitor Lambda が IAM モードのユーザーの使用量を正しく集計できる
- [ ] 既存の JWT モードの動作に影響がない
