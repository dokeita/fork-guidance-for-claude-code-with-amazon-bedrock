# Issue 003: IAM モードでのリアルタイムブロック強化

## 概要

IAM モードでのクォータ強制は Quota Monitor Lambda の15分間隔に依存しており、超過からブロックまで最大15分のギャップがある。よりリアルタイムな強制メカニズムが必要。

## 背景（Why）

- JWT モードでは credential-process が認証情報発行時に Quota Check API を呼び、即座にブロックできる
- IAM モードでは認証情報の発行を ccwb が制御しないため、同じメカニズムが使えない
- Quota Monitor Lambda は15分間隔で実行されるため、超過検知に最大15分の遅延がある
- 厳格なコスト管理が必要な組織では、この遅延が許容できない場合がある

## 対象（Who / What）

- **影響を受けるユーザー**: 厳格なクォータ制御が必要な組織の管理者
- **対象コンポーネント**: IAM ポリシー、Lambda、Bedrock アクセス制御

## 解決策の候補（How）

### 案 A: IAM ポリシーの動的更新によるブロック

クォータ超過時に IAM ポリシーに Deny ステートメントを追加し、Bedrock アクセスを直接ブロックする。

```
Quota Monitor Lambda → 超過検知
  → IAM Policy に Deny 追加: {"Effect": "Deny", "Action": "bedrock:*", "Resource": "*", "Condition": {"StringEquals": {"aws:username": "john.doe"}}}
  → ユーザーの次の API コールから即座にブロック
```

- **メリット**: 即座に効果あり、credential-process 不要
- **デメリット**: IAM ポリシーの変更頻度制限、ポリシーサイズ上限（6KB）、復旧の複雑さ

### 案 B: SCP (Service Control Policy) による組織レベルブロック

Organizations の SCP を動的に更新してブロック。

- **メリット**: アカウント横断で効果あり
- **デメリット**: Organizations 必須、SCP 変更の影響範囲が大きい

### 案 C: Bedrock のモデルアクセスポリシー活用

Bedrock のリソースベースポリシーやモデルアクセス設定を動的に変更。

- **メリット**: Bedrock に特化した制御
- **デメリット**: Bedrock のアクセス制御 API の制約に依存

### 案 D: VPC エンドポイントポリシーによるブロック

Bedrock VPC エンドポイントのポリシーを動的に更新。

- **メリット**: ネットワークレベルで確実にブロック
- **デメリット**: VPC エンドポイント経由のアクセスのみ対象、パブリックアクセスには効かない

## 実装場所（Where）

- `deployment/infrastructure/lambda-functions/quota_monitor/index.py`（ブロックアクション追加）
- `deployment/infrastructure/quota-monitoring.yaml`（IAM ポリシー操作権限）
- `source/claude_code_with_bedrock/cli/commands/quota.py`（unblock コマンドの IAM 対応）

## 優先度・時期（When）

- **優先度**: Medium
- **前提**: IAM モード対応の完了後、実運用でのフィードバック収集後
- **想定工数**: 5-10日（案 A の場合）

## 受け入れ条件

- [ ] クォータ超過から実際のブロックまでの遅延が1分以内
- [ ] ブロック解除（`ccwb quota unblock`）が正常に動作する
- [ ] IAM ポリシーサイズ上限を超えない設計（大規模ユーザー数対応）
- [ ] ブロック/解除の監査ログが CloudTrail に記録される

## リスク

- IAM ポリシーの変更は AWS API のレート制限に抵触する可能性がある
- ポリシー変更の伝播に数秒〜数十秒かかる場合がある
- 障害時にユーザーがブロックされたまま復旧できないリスク → fail-safe 設計が必須
