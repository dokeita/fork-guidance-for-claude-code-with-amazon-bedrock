# Issue 002: IAM モードでのブラウザ/ターミナル通知

## 概要

IAM モードでは credential-process バイナリを経由しないため、クォータ警告/ブロック時のブラウザ通知・ターミナル通知がエンドユーザーに届かない。

## 背景（Why）

- JWT モードでは credential-process がクォータチェック API を呼び、結果に応じてブラウザページを開いたりターミナルに警告を表示する
- IAM モードでは `aws sso login` や IAM User の認証情報を直接使うため、credential-process を経由しない
- 現状、IAM モードのユーザーはクォータ超過を SNS メール通知でしか知ることができない
- ユーザーが気づかないまま使い続け、突然ブロックされる体験が悪い

## 対象（Who / What）

- **影響を受けるユーザー**: IAM User / IAM Identity Center で認証している開発者
- **対象コンポーネント**: OTEL Helper、Claude Code 設定、通知メカニズム

## 解決策の候補（How）

### 案 A: OTEL Helper にクォータチェック機能を追加

OTEL Helper（メトリクス送信用のサイドカープロセス）にクォータチェックを組み込み、閾値超過時にデスクトップ通知を表示する。

```
OTEL Helper (常駐) → 定期的に Quota Check API を SigV4 で呼び出し
                    → 80%超過 → デスクトップ通知 (OS native notification)
                    → 100%超過 → デスクトップ通知 + ターミナル警告ファイル書き出し
```

- **メリット**: 既存の常駐プロセスを活用、追加インストール不要
- **デメリット**: OTEL Helper の責務が増える、sidecar モード前提

### 案 B: 専用のクォータ通知デーモン

軽量な常駐プロセスを新規作成し、定期的にクォータ状態を確認して通知する。

- **メリット**: 責務が明確、OTEL Helper と独立
- **デメリット**: 追加のプロセス管理が必要

### 案 C: Claude Code の hooks/MCP 連携

Claude Code の起動時フックや MCP サーバーとしてクォータ状態を提供する。

- **メリット**: Claude Code のエコシステムに統合
- **デメリット**: Claude Code の拡張ポイントに依存

## 実装場所（Where）

- `source/otel_helper/__main__.py`（案 A の場合）
- 新規デーモンスクリプト（案 B の場合）
- デスクトップ通知ライブラリ（OS 依存: macOS=osascript, Linux=notify-send, Windows=toast）

## 優先度・時期（When）

- **優先度**: Low
- **前提**: IAM モード対応（requirement.md セクション 4）の完了後
- **想定工数**: 5-7日

## 受け入れ条件

- [ ] IAM モードのユーザーがクォータ 80% 到達時にデスクトップ通知を受け取れる
- [ ] IAM モードのユーザーがクォータ 100% 到達時にブロック理由を視覚的に確認できる
- [ ] 通知が macOS / Linux / Windows で動作する
- [ ] 通知頻度が適切に制御されている（同一閾値で繰り返し通知しない）
