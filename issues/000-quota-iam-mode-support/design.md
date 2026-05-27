# Quota Monitoring 詳細設計書

## 1. 現行設計

### 1.1 システム構成図

```
┌─────────────────────────────────────────────────────────────────────────┐
│ エンドユーザー環境                                                        │
│                                                                         │
│  credential-process ──→ OIDC IdP ──→ id_token 取得                      │
│       │                                                                 │
│       ├──→ Quota Check API (Bearer: id_token)                           │
│       │         │                                                       │
│       │         ▼ allowed: true → STS AssumeRoleWithWebIdentity         │
│       │         ▼ allowed: false → ブラウザ通知 + ブロック               │
│       │                                                                 │
│  OTEL Helper ──→ JWT decode → user.email 抽出                           │
│       │                                                                 │
│       └──→ OTEL Collector (OTLP) → CloudWatch Metrics                   │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ AWS                                                                     │
│                                                                         │
│  ┌─────────────────────────────────────────┐                            │
│  │ API Gateway (HTTP API)                  │                            │
│  │  Route: GET /check                      │                            │
│  │  Auth: JWT Authorizer                   │                            │
│  │    Issuer: https://company.okta.com     │                            │
│  │    Audience: [client_id]                │                            │
│  └──────────────┬──────────────────────────┘                            │
│                 ▼                                                        │
│  ┌─────────────────────────────────────────┐                            │
│  │ Quota Check Lambda                      │                            │
│  │  1. JWT claims から email 取得          │                            │
│  │  2. QuotaPolicies から制限値取得        │                            │
│  │  3. UserQuotaMetrics から使用量取得     │                            │
│  │  4. 判定: allowed / blocked             │                            │
│  └─────────────────────────────────────────┘                            │
│                                                                         │
│  ┌─────────────────────────────────────────┐                            │
│  │ Quota Monitor Lambda (15分間隔)         │                            │
│  │  1. PromQL で使用量取得                 │                            │
│  │  2. UserQuotaMetrics に記録             │                            │
│  │  3. 閾値チェック                        │                            │
│  │  4. SNS アラート送信                    │                            │
│  └─────────────────────────────────────────┘                            │
│                                                                         │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐      │
│  │ UserQuotaMetrics │  │ QuotaPolicies    │  │ SNS Topic        │      │
│  │ (DynamoDB)       │  │ (DynamoDB)       │  │                  │      │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Quota Check API（現行: JWT モード）

#### リクエスト

```
GET /check
Authorization: Bearer <OIDC id_token>
```

#### 認証フロー

1. API Gateway JWT Authorizer がトークンを検証
2. Issuer URL と Audience (client_id) の一致を確認
3. 検証済み claims を Lambda に渡す

#### Lambda 内部処理

```python
# event.requestContext.authorizer.jwt.claims から取得
email = jwt_claims.get("email")
groups = extract_groups_from_claims(jwt_claims)  # groups, cognito:groups, custom:department
```

### 1.3 CloudFormation パラメータ（現行）

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| MonthlyTokenLimit | Number | ✓ | デフォルト月次制限 |
| WarningThreshold80 | Number | ✓ | 80% 閾値 |
| WarningThreshold90 | Number | ✓ | 90% 閾値 |
| DailyTokenLimit | Number | ✓ | デフォルト日次制限 |
| DailyEnforcementMode | String | ✓ | alert / block |
| MonthlyEnforcementMode | String | ✓ | alert / block |
| EnableFinegrainedQuotas | String | ✓ | true / false |
| OidcIssuerUrl | String | ✓ | OIDC Issuer URL |
| OidcClientId | String | ✓ | OIDC Client ID |

---

## 2. 改修後設計

### 2.1 システム構成図（IAM モード追加）

```
┌─────────────────────────────────────────────────────────────────────────┐
│ エンドユーザー環境 (IAM モード)                                          │
│                                                                         │
│  aws sso login / IAM User credentials                                   │
│       │                                                                 │
│  OTEL Helper ──→ STS GetCallerIdentity → user 名抽出                    │
│       │                                                                 │
│       └──→ OTEL Collector (OTLP) → CloudWatch Metrics                   │
│                user.email = <IAM User名 or SSO email>                   │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ AWS                                                                     │
│                                                                         │
│  ┌─────────────────────────────────────────┐                            │
│  │ API Gateway (HTTP API)                  │                            │
│  │  Route: GET /check                      │                            │
│  │  Auth: IAM (AWS_IAM)                    │  ← 変更点                  │
│  └──────────────┬──────────────────────────┘                            │
│                 ▼                                                        │
│  ┌─────────────────────────────────────────┐                            │
│  │ Quota Check Lambda                      │                            │
│  │  1. caller ARN からユーザー名抽出       │  ← 変更点                  │
│  │  2. QuotaPolicies から制限値取得        │                            │
│  │  3. UserQuotaMetrics から使用量取得     │                            │
│  │  4. 判定: allowed / blocked             │                            │
│  └─────────────────────────────────────────┘                            │
│                                                                         │
│  (Quota Monitor Lambda, DynamoDB, SNS は変更なし)                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 変更点サマリー

| コンポーネント | 現行 | 改修後 |
|---------------|------|--------|
| API Gateway Authorizer | JWT Authorizer (OIDC) | JWT or IAM（パラメータで切替） |
| Lambda ユーザー特定 | `jwt.claims.email` | JWT: 同左 / IAM: `requestContext.identity.userArn` |
| CFn パラメータ | OidcIssuerUrl, OidcClientId 必須 | AuthMode=iam 時は不要 |
| deploy.py | OidcIssuerUrl を常に渡す | sso_enabled=False 時は AuthMode=iam |
| init.py | SSO 無効時はクォータ不可 | SSO 無効でもクォータ有効化可能 |

---

## 3. 詳細設計

### 3.1 CloudFormation テンプレート変更

#### 追加パラメータ

```yaml
Parameters:
  AuthMode:
    Type: String
    Default: 'jwt'
    AllowedValues:
      - 'jwt'
      - 'iam'
    Description: >-
      Authentication mode for Quota Check API.
      jwt: OIDC JWT Authorizer (requires OidcIssuerUrl/OidcClientId).
      iam: IAM authorization via SigV4 (no OIDC required).
```

#### OidcIssuerUrl / OidcClientId の条件化

```yaml
  OidcIssuerUrl:
    Type: String
    Default: ''
    Description: OIDC provider issuer URL (required when AuthMode=jwt)

  OidcClientId:
    Type: String
    Default: ''
    Description: OIDC client ID (required when AuthMode=jwt)
```

#### Conditions

```yaml
Conditions:
  IsJwtMode: !Equals [!Ref AuthMode, 'jwt']
  IsIamMode: !Equals [!Ref AuthMode, 'iam']
```

#### Authorizer リソース（条件分岐）

```yaml
  # JWT Authorizer (JWT モードのみ)
  QuotaCheckJwtAuthorizer:
    Type: AWS::ApiGatewayV2::Authorizer
    Condition: IsJwtMode
    Properties:
      ApiId: !Ref QuotaCheckApi
      AuthorizerType: JWT
      Name: oidc-jwt-authorizer
      IdentitySource:
        - "$request.header.Authorization"
      JwtConfiguration:
        Issuer: !Ref OidcIssuerUrl
        Audience:
          - !Ref OidcClientId
```

#### Route（条件分岐）

```yaml
  # JWT 認証ルート
  QuotaCheckRouteJwt:
    Type: AWS::ApiGatewayV2::Route
    Condition: IsJwtMode
    Properties:
      ApiId: !Ref QuotaCheckApi
      RouteKey: 'GET /check'
      Target: !Sub 'integrations/${QuotaCheckIntegration}'
      AuthorizerId: !Ref QuotaCheckJwtAuthorizer
      AuthorizationType: JWT

  # IAM 認証ルート
  QuotaCheckRouteIam:
    Type: AWS::ApiGatewayV2::Route
    Condition: IsIamMode
    Properties:
      ApiId: !Ref QuotaCheckApi
      RouteKey: 'GET /check'
      Target: !Sub 'integrations/${QuotaCheckIntegration}'
      AuthorizationType: AWS_IAM
```

#### Lambda 環境変数追加

```yaml
  QuotaCheckFunction:
    Properties:
      Environment:
        Variables:
          AUTH_MODE: !Ref AuthMode
          # 既存の変数は維持
```

#### IAM モード用の呼び出し元ポリシー（Output）

```yaml
  QuotaCheckInvokePolicy:
    Condition: IsIamMode
    Description: IAM policy to allow invoking the Quota Check API (attach to user/role)
    Value: !Sub 'arn:${AWS::Partition}:execute-api:${AWS::Region}:${AWS::AccountId}:${QuotaCheckApi}/*/GET/check'
    Export:
      Name: !Sub '${AWS::StackName}-QuotaCheckInvokeArn'
```

### 3.2 Quota Check Lambda 変更

#### ユーザー特定ロジックの追加

```python
AUTH_MODE = os.environ.get("AUTH_MODE", "jwt")


def extract_user_identity(event: dict) -> tuple[str | None, list]:
    """
    認証モードに応じてユーザー identity を抽出する。

    Returns:
        (email_or_username, groups)
    """
    if AUTH_MODE == "jwt":
        # 現行ロジック: JWT claims から取得
        authorizer_context = event.get("requestContext", {}).get("authorizer", {})
        jwt_claims = authorizer_context.get("jwt", {}).get("claims", {})
        email = jwt_claims.get("email")
        groups = extract_groups_from_claims(jwt_claims)
        return email, groups

    elif AUTH_MODE == "iam":
        # 新規: caller ARN から取得
        identity = event.get("requestContext", {}).get("identity", {})
        user_arn = identity.get("userArn", "")
        username = parse_username_from_arn(user_arn)
        # IAM モードではグループは空（将来: DynamoDB マッピングで解決）
        return username, []

    return None, []


def parse_username_from_arn(arn: str) -> str | None:
    """
    IAM ARN からユーザー名を抽出する。

    パターン:
      arn:aws:iam::123456789012:user/john.doe
        → "john.doe"
      arn:aws:iam::123456789012:user/path/to/john.doe
        → "john.doe"
      arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_PermSet_abc123/john@company.com
        → "john@company.com"
      arn:aws:sts::123456789012:assumed-role/MyRole/session-name
        → "MyRole/session-name"
      arn:aws:iam::123456789012:root
        → "root"
    """
    if not arn:
        return None

    # arn:partition:service:region:account:resource
    parts = arn.split(":")
    if len(parts) < 6:
        return None

    resource = parts[5]  # e.g., "user/john.doe" or "assumed-role/Role/session"

    if resource.startswith("user/"):
        # IAM User: 最後の / 以降がユーザー名（パス付きの場合）
        return resource.split("/")[-1]

    elif resource.startswith("assumed-role/"):
        # assumed-role/RoleName/SessionName
        segments = resource.split("/")
        if len(segments) >= 3:
            role_name = segments[1]
            session_name = segments[2]

            # IAM Identity Center: AWSReservedSSO_* の場合、session がメールアドレス
            if role_name.startswith("AWSReservedSSO_"):
                return session_name  # 通常はメールアドレス

            return f"{role_name}/{session_name}"

    elif resource == "root":
        return "root"

    return resource
```

#### lambda_handler の変更

```python
def lambda_handler(event, context):
    try:
        # 変更: 認証モードに応じたユーザー特定
        email, groups = extract_user_identity(event)

        if not email:
            print(f"Could not identify user. AUTH_MODE={AUTH_MODE}")
            allow = MISSING_EMAIL_ENFORCEMENT != "block"
            return build_response(200, {
                "allowed": allow,
                "reason": "missing_identity",
                "message": f"Could not identify user ({AUTH_MODE} mode)"
            })

        # 以降は現行ロジックと同一
        policy = resolve_quota_for_user(email, groups)
        # ...
```

### 3.3 deploy.py 変更

```python
# 現行 (740行目付近)
elif stack_type == "quota":
    # ...

    # 変更: AuthMode の判定
    sso_enabled = getattr(profile, "sso_enabled", True)
    auth_mode = "jwt" if sso_enabled else "iam"

    if auth_mode == "jwt":
        # 現行ロジック: OIDC 設定を取得
        if profile.provider_type == "cognito":
            # ... (既存の cognito 処理)
        else:
            oidc_issuer_url = profile.provider_domain
            if oidc_issuer_url and not oidc_issuer_url.startswith(("http://", "https://")):
                oidc_issuer_url = f"https://{oidc_issuer_url}"
        oidc_client_id = profile.client_id
    else:
        # IAM モード: OIDC パラメータ不要
        oidc_issuer_url = ""
        oidc_client_id = ""

    params = [
        f"AuthMode={auth_mode}",  # 追加
        f"MonthlyTokenLimit={monthly_limit}",
        f"OidcIssuerUrl={oidc_issuer_url}",
        f"OidcClientId={oidc_client_id}",
        # ... 他のパラメータ
    ]
```

### 3.4 init.py 変更

```python
# 現行: SSO 無効時はクォータ監視の選択肢を表示しない
# 改修: SSO 無効でもクォータ監視を有効化可能にする

# 変更箇所: Optional Features セクションのクォータ監視プロンプト
# 条件 "if config.get('sso_enabled', True):" を削除し、常に表示
```

---

## 4. データフロー比較

### 4.1 JWT モード（変更なし）

```
credential-process
  → ブラウザ OIDC 認証 → id_token 取得
  → GET /check (Authorization: Bearer <id_token>)
  → API Gateway JWT Authorizer 検証
  → Lambda: jwt.claims.email = "john@company.com"
  → DynamoDB 参照 → allowed/blocked
  → credential-process がブラウザ通知 or 認証情報発行
```

### 4.2 IAM モード（新規）

```
Quota Monitor Lambda (15分間隔)
  → PromQL: user.email 別の使用量取得
  → UserQuotaMetrics 更新
  → 閾値チェック → SNS アラート

(オプション: 外部クライアントからの直接チェック)
  → GET /check (SigV4 署名)
  → API Gateway IAM Authorizer 検証
  → Lambda: requestContext.identity.userArn パース
  → DynamoDB 参照 → allowed/blocked
```

---

## 5. API インターフェース

### 5.1 Quota Check API

#### エンドポイント

```
GET /check
```

#### 認証

| モード | ヘッダー | 検証 |
|--------|----------|------|
| JWT | `Authorization: Bearer <id_token>` | API Gateway JWT Authorizer |
| IAM | SigV4 署名ヘッダー群 | API Gateway IAM Authorizer |

#### レスポンス（共通、変更なし）

```json
{
  "allowed": true,
  "reason": "within_quota",
  "enforcement_mode": "block",
  "usage": {
    "monthly_tokens": 150000000,
    "monthly_limit": 225000000,
    "monthly_percentage": 66.7,
    "daily_tokens": 5000000,
    "daily_limit": 8250000,
    "daily_percentage": 60.6
  },
  "policy": {
    "type": "default",
    "identifier": "default"
  },
  "unblock_status": {
    "is_unblocked": false
  },
  "message": "Access granted - within quota limits"
}
```

---

## 6. DynamoDB スキーマ（変更なし）

### UserQuotaMetrics

| PK | SK | 用途 |
|----|-----|------|
| `USER#<email_or_username>` | `MONTH#YYYY-MM` | 月次使用量 |
| `USER#<email_or_username>` | `UNBLOCK#CURRENT` | ブロック解除状態 |
| `ALERTS` | `{month}#ALERT#{user}#{type}#{level}` | アラート送信履歴 |

### QuotaPolicies

| PK | SK | 用途 |
|----|-----|------|
| `POLICY#user#<email_or_username>` | `CURRENT` | ユーザーポリシー |
| `POLICY#group#<group_name>` | `CURRENT` | グループポリシー |
| `POLICY#default#default` | `CURRENT` | デフォルトポリシー |

**注意**: IAM モードでは `email_or_username` が IAM User 名（例: `john.doe`）になる。ポリシー設定時も同じ識別子を使う。

---

## 7. セキュリティ設計

### 7.1 JWT モード（変更なし）

- API Gateway が OIDC トークンを検証（署名、有効期限、Issuer、Audience）
- Lambda は検証済み claims のみを参照（改ざん不可）

### 7.2 IAM モード（新規）

- API Gateway が SigV4 署名を検証（AWS IAM による認証）
- `requestContext.identity.userArn` は AWS が設定（改ざん不可）
- 呼び出し元に `execute-api:Invoke` 権限が必要（IAM ポリシーで制御）

### 7.3 IAM モードの呼び出し元ポリシー

Quota Check API を呼び出す IAM User/Role に以下のポリシーをアタッチ：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:<region>:<account>:<api-id>/*/GET/check"
    }
  ]
}
```

---

## 8. テスト計画

### 8.1 Quota Check Lambda

| テストケース | 入力 | 期待結果 |
|-------------|------|----------|
| IAM User でクォータ内 | SigV4 署名、使用量 < 制限 | `allowed: true` |
| IAM User でクォータ超過 | SigV4 署名、使用量 ≥ 制限 | `allowed: false, reason: monthly_exceeded` |
| IAM Identity Center ユーザー | AWSReservedSSO_* ARN | email が正しく抽出される |
| AssumeRole ユーザー | assumed-role ARN | `role/session` 形式で識別 |
| ARN パース失敗 | 不正な ARN | fail-open/closed 設定に従う |
| JWT モード（回帰テスト） | Bearer token | 現行動作と同一 |
| AuthMode=jwt で OIDC 未設定 | デプロイ時 | CFn バリデーションエラー |
| AuthMode=iam で OIDC 未設定 | デプロイ時 | 正常デプロイ |

### 8.2 OTEL Helper（Issue 004）

| テストケース | 入力 | 期待結果 |
|-------------|------|----------|
| IAM User | ARN `arn:aws:iam::*:user/john.doe` | `x-user-email: john.doe` |
| IAM User (パス付き) | ARN `arn:aws:iam::*:user/devs/john.doe` | `x-user-email: john.doe` |
| IAM Identity Center | ARN `...AWSReservedSSO_.../john@co.com` | `x-user-email: john@co.com` |
| 非SSO AssumeRole | ARN `...assumed-role/MyRole/ci-build` | `x-user-email: MyRole/ci-build` |
| JWT モード（回帰） | 有効な JWT トークン | 現行動作と同一（email claim 使用） |
| 識別子一致確認 | IAM User "john.doe" | OTEL, Quota Monitor, Quota Check 全てで `john.doe` |

### 8.3 E2E テスト

| テストケース | 手順 | 期待結果 |
|-------------|------|----------|
| IAM User のクォータ追跡 | 1. IAM User で Bedrock 使用 → 2. 15分待機 → 3. `ccwb quota usage john.doe` | 使用量が記録されている |
| IAM User のブロック | 1. 制限を低く設定 → 2. 使用 → 3. Quota Check API 呼び出し | `allowed: false` |
| JWT→IAM 移行 | 1. JWT モードで運用中 → 2. AuthMode=iam に変更 → 3. 再デプロイ | 既存データ維持、新モードで動作 |

---

## 9. OTEL Helper のユーザー特定（Issue 004 対応）

### 9.1 現行動作

OTEL Helper (`otel_helper/__main__.py`) の `get_user_headers()` は既に以下のフォールバックを実装している：

```python
def get_user_headers():
    token = None
    if not ANONYMOUS_MODE:
        token = os.environ.get("CLAUDE_CODE_MONITORING_TOKEN") or get_token_via_credential_process()

    if token:
        # JWT モード: トークンから email を抽出
        payload = decode_jwt_payload(token)
        user_info = extract_user_info(payload)
    else:
        # IAM モード: STS GetCallerIdentity から ARN を取得してパース
        caller_identity = get_aws_caller_identity()
        user_info = create_anonymous_user_info(caller_identity)

    return format_as_headers_dict(user_info)
```

### 9.2 既存の ARN パースロジック（`_parse_arn_identity`）

| ARN パターン | 抽出結果 | email ラベル |
|-------------|----------|-------------|
| `arn:aws:iam::*:user/<username>` | username | `<username>@anonymous` |
| `arn:aws:iam::*:user/<path>/<username>` | username (最後のセグメント) | `<username>@anonymous` |
| `arn:aws:sts::*:assumed-role/AWSReservedSSO_*/<session>` | session (通常はメール) | session そのまま（`@` 含む場合） |
| `arn:aws:sts::*:assumed-role/<Role>/<Session>` (非SSO) | session_name | `<session_name>@anonymous` |

### 9.3 問題点と改修

現行の OTEL Helper は IAM モードで `<username>@anonymous` 形式の email を生成する。一方、Quota Check Lambda は `requestContext.identity.userArn` から `<username>` を抽出する。

**不整合**: OTEL メトリクスの `user.email` = `john.doe@anonymous` だが、Quota Check Lambda が識別するユーザー = `john.doe`。Quota Monitor Lambda が PromQL で取得する使用量のキーと、QuotaPolicies のキーが一致しない。

#### 改修方針

OTEL Helper と Quota Check Lambda のユーザー識別子を統一する。

| コンポーネント | 現行 | 改修後 |
|---------------|------|--------|
| OTEL Helper (IAM User) | `john.doe@anonymous` | `john.doe` |
| OTEL Helper (SSO) | `john@company.com` | `john@company.com`（変更なし） |
| OTEL Helper (非SSO AssumeRole) | `session-name@anonymous` | `<role>/<session>` |
| Quota Check Lambda (IAM User) | — | `john.doe` |
| Quota Check Lambda (SSO) | — | `john@company.com` |
| Quota Monitor Lambda (PromQL) | `user.email` で集計 | 同左（値が統一されるため動作する） |
| QuotaPolicies (CLI 設定) | — | `ccwb quota set-user john.doe` |

#### 具体的な変更

**`otel_helper/__main__.py` — `_parse_arn_identity` の email フィールド修正:**

```python
# Case 2: IAM user
if resource.startswith("user/"):
    username = user_path.rsplit("/", 1)[-1]
    return {
        "username": username,
        "email": username,  # 変更: "@anonymous" を付けない
        "role": "iam-user",
        "issuer": "aws-iam",
    }
```

**`otel_helper/__main__.py` — `create_anonymous_user_info` の非SSO AssumeRole:**

```python
# 非SSO assumed role
if role_info:
    identifier = f"{role_info['role_name']}/{role_info['session_name']}"
    return {
        "email": identifier,  # 変更: "@anonymous" を付けない
        "user_id": role_info["session_name"],
        "username": role_info["session_name"],
        # ...
    }
```

**`quota_check/index.py` — `parse_username_from_arn` は design.md セクション 3.2 の通り。**

### 9.4 識別子の一致確認

改修後、全コンポーネントで同じ識別子が使われることを確認：

```
IAM User "john.doe" の場合:

  OTEL Helper:
    x-user-email: john.doe
    → CloudWatch メトリクス: user.email="john.doe"

  Quota Monitor Lambda:
    PromQL: sum by ("user.email")(increase(...))
    → email="john.doe", total_tokens=X
    → DynamoDB: PK=USER#john.doe, SK=MONTH#2026-05

  Quota Check Lambda:
    ARN: arn:aws:iam::123456789012:user/john.doe
    → parse_username_from_arn → "john.doe"
    → DynamoDB lookup: PK=USER#john.doe

  ccwb quota set-user:
    ccwb quota set-user john.doe --monthly-limit 225M
    → DynamoDB: PK=POLICY#user#john.doe, SK=CURRENT
```

### 9.5 後方互換性

| 懸念 | 対応 |
|------|------|
| 既存の `@anonymous` 付きデータが DynamoDB に残る | TTL で自動削除（翌月末）。移行期間中は旧データと新データが共存するが、新データが優先される |
| JWT モードの email に影響 | JWT モードは `extract_user_info(payload)` を使うため変更なし |
| ダッシュボードの表示 | `user.email` ラベルの値が変わるため、移行月は旧名と新名が別ユーザーとして表示される |

---

## 10. 移行・互換性

| 項目 | 対応 |
|------|------|
| 既存 JWT モードデプロイ | `AuthMode` デフォルト = `jwt`。変更なしで動作継続 |
| 既存 UserQuotaMetrics データ | スキーマ変更なし。IAM ユーザー名でも同じ PK 構造 |
| 既存 QuotaPolicies | そのまま利用可能。IAM User 名でポリシー設定 |
| ccwb quota CLI | `--user` に IAM User 名を指定可能（email 形式でなくてもよい） |
