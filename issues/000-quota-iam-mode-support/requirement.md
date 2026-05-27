# Quota Monitoring 要件定義

## 1. 概要

Claude Code with Bedrock の Quota Monitoring 機能について、現行機能の維持と IAM 認証モード対応の改修を定義する。

---

## 2. 認証モード

Quota Monitoring は以下の2つの認証モードをサポートする。

| モード | API 認証方式 | ユーザー特定方法 | 対象ユースケース |
|--------|-------------|-----------------|-----------------|
| **JWT モード** (現行) | API Gateway JWT Authorizer | JWT claims の `email` | External IdP (Okta/Azure AD/Auth0/Cognito) |
| **IAM モード** (新規) | API Gateway IAM Authorizer (SigV4) | caller ARN からユーザー名抽出 | IAM User / IAM Identity Center / AssumeRole |

### 2.1 IAM モードのユーザー特定ルール

| caller ARN パターン | 抽出結果 |
|---------------------|----------|
| `arn:aws:iam::*:user/<username>` | `<username>` |
| `arn:aws:sts::*:assumed-role/AWSReservedSSO_*/<email>` | `<email>` |
| `arn:aws:sts::*:assumed-role/<role>/<session>` | `<role>/<session>` |

---

## 3. 機能一覧

### 3.1 使用量追跡

| ID | 機能 | 説明 | JWT | IAM |
|----|------|------|-----|-----|
| T-1 | 月次トークン追跡 | ユーザーごとの月次トークン消費量を DynamoDB に記録 | ✓ | ✓ |
| T-2 | 日次トークン追跡 | ユーザーごとの日次トークン消費量を記録（UTC 0時リセット） | ✓ | ✓ |
| T-3 | トークン種別内訳 | input / output / cache_read の内訳を記録 | ✓ | ✓ |
| T-4 | PromQL データ取得 | CloudWatch Prometheus API から使用量を15分間隔で取得 | ✓ | ✓ |
| T-5 | 月次自動リセット | 毎月1日に月次カウンターをリセット（TTL による自動削除） | ✓ | ✓ |

### 3.2 クォータポリシー

| ID | 機能 | 説明 | JWT | IAM |
|----|------|------|-----|-----|
| P-1 | デフォルトポリシー | 全ユーザーに適用される基本制限 | ✓ | ✓ |
| P-2 | ユーザーポリシー | 特定ユーザーへの個別制限（最優先） | ✓ | ✓ |
| P-3 | グループポリシー | グループ単位の制限（JWT claims の groups から取得） | ✓ | ✗ (*1) |
| P-4 | ポリシー優先順位 | ユーザー > グループ > デフォルト | ✓ | ✓ |
| P-5 | 月次制限 | 月あたりの最大トークン数 | ✓ | ✓ |
| P-6 | 日次制限 | 日あたりの最大トークン数（バーストバッファ付き） | ✓ | ✓ |

> *1: IAM モードでは JWT claims が存在しないため、グループポリシーは CLI で手動割り当て（将来拡張）

### 3.3 強制（Enforcement）

| ID | 機能 | 説明 | JWT | IAM |
|----|------|------|-----|-----|
| E-1 | alert モード | 閾値超過時に SNS 通知のみ、アクセス継続可 | ✓ | ✓ |
| E-2 | block モード | 閾値超過時にクレデンシャル発行を拒否 | ✓ | ✓ |
| E-3 | リアルタイムクォータチェック | 認証時に Quota Check API を呼び出し、許可/拒否を判定 | ✓ | ✓ |
| E-4 | 定期再チェック | キャッシュ済み認証情報使用中も定期的にクォータ確認（デフォルト30分） | ✓ | ✗ (*2) |
| E-5 | fail-open / fail-closed | API エラー時の動作を設定可能 | ✓ | ✓ |

> *2: IAM モードでは credential-process を経由しないため、定期再チェックは対象外。Quota Monitor Lambda の15分間隔チェック + SNS アラートで代替。

### 3.4 アラート

| ID | 機能 | 説明 | JWT | IAM |
|----|------|------|-----|-----|
| A-1 | 80% 警告 | 月次使用量が80%到達時に SNS 通知 | ✓ | ✓ |
| A-2 | 90% 警告 | 月次使用量が90%到達時に SNS 通知 | ✓ | ✓ |
| A-3 | 100% 超過 | 月次使用量が100%到達時に SNS 通知 | ✓ | ✓ |
| A-4 | 日次超過アラート | 日次制限超過時に SNS 通知 | ✓ | ✓ |
| A-5 | アラート重複排除 | 同一閾値・同一期間で1回のみ通知（DynamoDB TTL 60日） | ✓ | ✓ |
| A-6 | ブラウザ通知 | 警告/ブロック時にブラウザでステータス表示 | ✓ | ✗ (*3) |
| A-7 | ターミナル通知 | 警告/ブロック時にターミナルに表示 | ✓ | ✗ (*3) |

> *3: IAM モードでは credential-process を経由しないため、エンドユーザーへの直接通知は SNS 経由（メール等）で代替。

### 3.5 管理 CLI (`ccwb quota`)

| ID | 機能 | 説明 | JWT | IAM |
|----|------|------|-----|-----|
| C-1 | `set-default` | デフォルトポリシー設定 | ✓ | ✓ |
| C-2 | `set-user` | ユーザー別ポリシー設定 | ✓ | ✓ |
| C-3 | `set-group` | グループ別ポリシー設定 | ✓ | ✓ |
| C-4 | `list` | ポリシー一覧表示 | ✓ | ✓ |
| C-5 | `usage` | ユーザー使用量確認 | ✓ | ✓ |
| C-6 | `unblock` | ブロック解除（一時的） | ✓ | ✓ |
| C-7 | `export` | ポリシーを JSON/CSV エクスポート | ✓ | ✓ |
| C-8 | `import` | ポリシーを JSON/CSV インポート | ✓ | ✓ |

### 3.6 インフラストラクチャ

| ID | 機能 | 説明 | JWT | IAM |
|----|------|------|-----|-----|
| I-1 | DynamoDB (UserQuotaMetrics) | 使用量記録テーブル | ✓ | ✓ |
| I-2 | DynamoDB (QuotaPolicies) | ポリシー定義テーブル | ✓ | ✓ |
| I-3 | Quota Monitor Lambda | 15分間隔の使用量チェック + アラート送信 | ✓ | ✓ |
| I-4 | Quota Check API (JWT) | API Gateway + JWT Authorizer | ✓ | ✗ |
| I-5 | Quota Check API (IAM) | API Gateway + IAM Authorizer | ✗ | ✓ |
| I-6 | SNS Topic | アラート配信 | ✓ | ✓ |
| I-7 | EventBridge Rule | Lambda スケジューリング | ✓ | ✓ |

---

## 4. 改修スコープ

### 4.1 CloudFormation テンプレート (`quota-monitoring.yaml`)

| 変更 | 内容 |
|------|------|
| パラメータ追加 | `AuthMode` (jwt / iam) — デフォルト: jwt |
| パラメータ条件化 | `OidcIssuerUrl`, `OidcClientId` を JWT モード時のみ必須に |
| Condition 追加 | `IsJwtMode`, `IsIamMode` |
| Authorizer 分岐 | JWT モード → JWT Authorizer / IAM モード → IAM 認証 (AWS_IAM) |
| IAM ポリシー追加 | IAM モード時、呼び出し元に `execute-api:Invoke` 権限を付与するポリシー出力 |

### 4.2 Quota Check Lambda (`quota_check/index.py`)

| 変更 | 内容 |
|------|------|
| ユーザー特定ロジック追加 | IAM モード時は `event.requestContext.identity.userArn` からユーザー名を抽出 |
| 環境変数追加 | `AUTH_MODE` (jwt / iam) |
| フォールバック | ARN パース失敗時は fail-open/fail-closed 設定に従う |

### 4.3 CLI (`deploy.py`)

| 変更 | 内容 |
|------|------|
| AuthMode 判定 | `sso_enabled=False` の場合 `AuthMode=iam` を渡す |
| パラメータ省略 | IAM モード時は `OidcIssuerUrl`, `OidcClientId` を渡さない |

### 4.4 CLI (`init.py`)

| 変更 | 内容 |
|------|------|
| クォータ有効化条件の緩和 | SSO 無効でもクォータ監視を有効にできるようにする |

---

## 5. 非機能要件

| 項目 | 要件 |
|------|------|
| レイテンシ | Quota Check API のレスポンス: < 500ms (p99) |
| 可用性 | Quota Monitor Lambda 失敗時もユーザーアクセスに影響なし（fail-open デフォルト） |
| コスト | IAM モードでも追加コストは既存と同等（$2-10/月、<1000ユーザー） |
| 後方互換性 | 既存の JWT モードデプロイに影響なし。`AuthMode` デフォルトは `jwt` |
| セキュリティ | IAM モードでは SigV4 署名により呼び出し元を認証。ARN の改ざん不可 |

---

## 6. 制約事項

| 制約 | 理由 |
|------|------|
| IAM モードではグループポリシーが自動適用されない | JWT claims が存在しないため。CLI で手動割り当てが必要 |
| IAM モードではブラウザ/ターミナル通知なし | credential-process を経由しないため |
| IAM モードでの強制は15分間隔 | Quota Monitor Lambda のスケジュールに依存。リアルタイムブロックは Quota Check API を呼ぶクライアント実装が別途必要 |
| ユーザー特定は OTEL メトリクスの `user.email` ラベルに依存 | モニタリングスタック（OTEL Collector）が正しくユーザーを識別している前提 |

---

## 7. 将来拡張（スコープ外）

- IAM モードでのグループ自動割り当て（IAM タグや Organizations OU ベース）
- Bedrock 側の Usage API との連携による credential-process 非依存のリアルタイムブロック
- Slack/Teams 直接通知（SNS → Lambda → Webhook）
- 管理者 Web UI（クォータダッシュボード）
