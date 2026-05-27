# Issue 005: Slack/Teams 直接通知

## 概要

クォータアラートの通知先を SNS メール以外に拡張し、Slack や Microsoft Teams のチャンネルに直接通知できるようにする。

## 背景（Why）

- 現状のアラート配信は SNS Topic 経由のメール通知のみ
- 多くの組織では Slack や Teams が主要なコミュニケーションツールであり、メール通知は見落とされやすい
- 管理者だけでなく、ユーザー本人にも DM で通知したいニーズがある
- SNS → Lambda → Webhook の構成は可能だが、ユーザーが自分で構築する必要がある

## 対象（Who / What）

- **影響を受けるユーザー**: Slack/Teams を利用している組織の管理者・開発者
- **対象コンポーネント**: SNS Topic、通知 Lambda、ccwb init 設定

## 解決策（How）

### アーキテクチャ

```
Quota Monitor Lambda → SNS Topic → Notification Lambda → Slack/Teams Webhook
                                                       → ユーザー DM (Slack API)
```

### 通知チャネル

| チャネル | 用途 | 設定方法 |
|----------|------|----------|
| Slack Incoming Webhook | 管理者チャンネルへの一括通知 | Webhook URL を設定 |
| Slack Bot (API) | ユーザー本人への DM | Bot Token + email→Slack ID マッピング |
| Teams Incoming Webhook | 管理者チャンネルへの一括通知 | Webhook URL を設定 |
| Teams Bot | ユーザー本人への DM | Bot 登録 + Graph API |

### 通知フォーマット例（Slack）

```json
{
  "blocks": [
    {
      "type": "header",
      "text": {"type": "plain_text", "text": "⚠️ Claude Code Quota Warning"}
    },
    {
      "type": "section",
      "fields": [
        {"type": "mrkdwn", "text": "*User:*\njohn.doe@company.com"},
        {"type": "mrkdwn", "text": "*Usage:*\n180M / 225M tokens (80%)"},
        {"type": "mrkdwn", "text": "*Period:*\nMay 2026"},
        {"type": "mrkdwn", "text": "*Policy:*\ngroup:engineering"}
      ]
    }
  ]
}
```

## 実装場所（Where）

- `deployment/infrastructure/lambda-functions/quota_notification/index.py`（新規）
- `deployment/infrastructure/quota-monitoring.yaml`（Lambda + SNS Subscription 追加）
- `source/claude_code_with_bedrock/cli/commands/init.py`（Webhook URL 設定プロンプト）
- `source/claude_code_with_bedrock/config.py`（設定モデル追加）

## 優先度・時期（When）

- **優先度**: Low
- **前提**: 基本的な Quota Monitoring が安定稼働していること
- **想定工数**: 3-5日

## 受け入れ条件

- [ ] `ccwb init` で Slack/Teams Webhook URL を設定できる
- [ ] クォータ閾値超過時に指定チャンネルに通知が届く
- [ ] 通知フォーマットが Slack/Teams のリッチメッセージ形式で表示される
- [ ] Webhook URL は AWS Secrets Manager に保存される（平文で設定ファイルに残さない）
- [ ] Webhook 送信失敗時にリトライされる（DLQ 付き）
- [ ] 既存の SNS メール通知と併用可能
