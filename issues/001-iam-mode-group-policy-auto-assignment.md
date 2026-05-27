# Issue 001: IAM モードでのグループポリシー自動適用

## 概要

IAM モードでは JWT claims が存在しないため、グループポリシーが自動適用されない。現状は CLI で手動割り当てが必要であり、大規模組織での運用負荷が高い。

## 背景（Why）

- JWT モードでは OIDC トークンの `groups` / `cognito:groups` / `custom:department` claims からグループメンバーシップを自動抽出し、グループポリシーを適用している
- IAM モードでは認証情報が SigV4 署名のみであり、JWT claims が存在しない
- 管理者が `ccwb quota set-group` でグループを定義しても、ユーザーとグループの紐付けが自動化されていない

## 対象（Who / What）

- **影響を受けるユーザー**: IAM User / IAM Identity Center で認証している組織の管理者
- **対象コンポーネント**: Quota Check Lambda、QuotaPolicies テーブル、ccwb CLI

## 解決策の候補（How）

### 案 A: IAM タグベースのグループ割り当て

IAM User のタグ（例: `Department=engineering`）を読み取り、グループポリシーにマッピングする。

```
IAM User Tag: Department=engineering
→ グループポリシー "engineering" を適用
```

- **メリット**: IAM の既存機能を活用、IdP 連携不要
- **デメリット**: IAM User にタグ付けが必要、AssumeRole 経由だとタグが見えない場合がある

### 案 B: DynamoDB にユーザー→グループマッピングテーブルを追加

管理者が CLI でユーザーとグループの紐付けを登録する。

```bash
ccwb quota assign-group john.doe engineering
ccwb quota assign-group jane.doe ml-team
```

- **メリット**: シンプル、IAM 構成に依存しない
- **デメリット**: 手動管理が残る（ただし一括インポートで軽減可能）

### 案 C: AWS Organizations OU ベースのグループ割り当て

Organizations の OU 構造をグループとして利用する。

- **メリット**: 組織構造と自然に一致
- **デメリット**: Organizations 未使用の環境では利用不可

## 実装場所（Where）

- `deployment/infrastructure/lambda-functions/quota_check/index.py`
- `deployment/infrastructure/quota-monitoring.yaml`（IAM ポリシー追加）
- `source/claude_code_with_bedrock/cli/commands/quota.py`（CLI コマンド追加）

## 優先度・時期（When）

- **優先度**: Medium
- **前提**: IAM モード対応（requirement.md セクション 4）の完了後
- **想定工数**: 3-5日

## 受け入れ条件

- [ ] IAM モードでグループポリシーがユーザーに自動適用される
- [ ] 管理者がグループ割り当てを一括管理できる（import/export 対応）
- [ ] 既存の JWT モードのグループポリシー動作に影響がない
