---
layout: default
title: インフラ設計
---

# インフラ設計 — expense-mvp

## 1. AWSリソース一覧

### 1.1 コンピュート

| リソース | 仕様 | 環境別設定 |
|---------|------|-----------|
| Lambda: api-handler | Python 3.12、メモリ512MB、タイムアウト30秒 | dev: 128MB / staging: 256MB / prod: 512MB |
| Lambda: batch-hr-import | Python 3.12、メモリ512MB、タイムアウト900秒 | 全環境同一 |
| Lambda: batch-org-import | Python 3.12、メモリ256MB、タイムアウト300秒 | 全環境同一 |
| Lambda: batch-journal-export | Python 3.12、メモリ512MB、タイムアウト900秒 | 全環境同一 |
| Lambda Layer: common-deps | 共通Pythonライブラリ | 全環境同一 |

#### Lambda IAMロール

| ロール | 権限 |
|--------|------|
| expense-api-role | RDS接続、S3読み書き（attachmentsバケット）、SES送信、Secrets Manager読み取り、CloudWatch Logs書き込み |
| expense-batch-role | RDS接続、S3読み書き（batchバケット）、Secrets Manager読み取り、CloudWatch Logs書き込み |

#### Lambda環境変数

| 変数名 | 値 | 説明 |
|--------|-----|------|
| DB_SECRET_ARN | Secrets Manager ARN | DB接続情報のシークレットARN |
| S3_ATTACHMENT_BUCKET | バケット名 | 添付ファイルバケット |
| S3_BATCH_BUCKET | バケット名 | バッチファイルバケット |
| SES_SENDER_EMAIL | noreply@example.com | メール送信元 |
| OIDC_ISSUER | IdP発行者URL | OIDC Issuer |
| OIDC_AUDIENCE | クライアントID | OIDC Audience |
| OIDC_JWKS_URL | JWKS URL | JWKS エンドポイント |
| FRONTEND_URL | フロントエンドURL | CORS許可オリジン、メールリンク |
| LOG_LEVEL | INFO | ログレベル |
| ENVIRONMENT | dev/staging/production | 環境識別子 |

### 1.2 API Gateway

| 項目 | 設定 |
|------|------|
| タイプ | REST API |
| ステージ | dev / staging / production |
| 認証 | Lambda Authorizer（OIDC トークン検証） |
| スロットリング | 1,000 req/sec（バースト: 2,000） |
| アクセスログ | 有効（CloudWatch Logs） |
| CORS | フロントエンドオリジンのみ許可 |
| バイナリメディアタイプ | multipart/form-data（ファイルアップロード用） |
| リクエストバリデーション | 有効（リクエストボディ） |
| 使用量プラン | 認証エンドポイント: 10 req/min/IP |

### 1.3 データベース（RDS）

| 項目 | dev | staging | production |
|------|-----|---------|------------|
| エンジン | PostgreSQL 16 | PostgreSQL 16 | PostgreSQL 16 |
| インスタンスクラス | db.t3.micro | db.t3.small | db.t3.medium |
| ストレージ | gp3、20GB | gp3、20GB | gp3、20GB（自動拡張100GB） |
| マルチAZ | 無効 | 無効 | 有効 |
| 自動バックアップ | 1日（保持1日） | 1日（保持3日） | 1時間（保持7日） |
| 暗号化 | 有効 | 有効 | 有効 |
| パフォーマンスインサイト | 無効 | 有効 | 有効 |
| 削除保護 | 無効 | 無効 | 有効 |

#### RDSパラメータグループ（カスタム）

| パラメータ | 値 | 説明 |
|-----------|-----|------|
| log_statement | all | 全SQLログ出力（dev/staging） |
| log_min_duration_statement | 1000 | 1秒以上のスロークエリログ（production） |
| shared_buffers | 128MB | バッファサイズ（db.t3.medium） |
| max_connections | 100 | 最大接続数 |

### 1.4 ストレージ（S3）

| バケット（論理名） | 暗号化 | バージョニング | パブリックアクセス | ライフサイクル |
|------------------|--------|-------------|-----------------|-------------|
| {env}-expense-attachments | SSE-S3 | 有効 | 全ブロック | 3年後Glacier、4年後削除 |
| {env}-expense-batch | SSE-S3 | 無効 | 全ブロック | 90日後削除 |
| {env}-expense-frontend | SSE-S3 | 有効 | CloudFront OAI経由のみ | なし |

#### S3バケットポリシー

| バケット | ポリシー |
|---------|---------|
| attachments | Lambda IAMロールからのみ読み書き許可 |
| batch | Lambda IAMロールからのみ読み書き許可 |
| frontend | CloudFront OAIからのみ読み取り許可 |

### 1.5 CDN（CloudFront）

| 項目 | 設定 |
|------|------|
| オリジン1 | S3（フロントエンド）— OAI経由 |
| オリジン2 | API Gateway — /api/* パスパターン |
| デフォルトルートオブジェクト | index.html |
| カスタムエラーページ | 403/404 → /index.html（SPA対応） |
| キャッシュポリシー | 静的アセット: 1日 / API: キャッシュなし |
| SSL証明書 | ACM（us-east-1） |
| 最小TLSバージョン | TLSv1.2_2021 |
| WAF | WebACL関連付け |
| ログ | S3バケットへアクセスログ出力 |

### 1.6 WAF

| ルール | 優先度 | アクション |
|--------|--------|----------|
| AWS-AWSManagedRulesCommonRuleSet | 1 | Block |
| AWS-AWSManagedRulesSQLiRuleSet | 2 | Block |
| AWS-AWSManagedRulesKnownBadInputsRuleSet | 3 | Block |
| RateLimit-Global | 4 | Block（2,000 req/5min/IP） |
| RateLimit-Auth | 5 | Block（50 req/5min/IP、/api/v1/auth/*） |

### 1.7 SES

| 項目 | 設定 |
|------|------|
| 送信元ドメイン | 要設定（DKIM、SPF、DMARC） |
| 送信元アドレス | noreply@{domain} |
| 設定セット | expense-notification（バウンス・苦情追跡） |
| サンドボックス | dev/staging: サンドボックスモード / production: 本番モード |

### 1.8 Secrets Manager

| シークレット名 | 内容 | ローテーション |
|-------------|------|-------------|
| {env}/expense/db-credentials | DB接続情報（host, port, dbname, username, password） | 90日（自動） |
| {env}/expense/oidc-config | IdP設定（client_id, client_secret, issuer, jwks_url） | 手動 |

### 1.9 モニタリング

| リソース | 設定 |
|---------|------|
| CloudWatch Logs | ロググループ5つ（API Gateway、Lambda×4） |
| CloudWatch Alarms | 7つ（API エラー率、レイテンシ、Lambda エラー、RDS CPU/接続数、認証/認可失敗） |
| CloudWatch Dashboard | 1つ（API概要、Lambda、RDS、バッチ） |
| SNS Topic | 1つ（アラーム通知先: 運用チームメール） |

### 1.10 EventBridge

| ルール名 | スケジュール | ターゲット |
|---------|------------|----------|
| expense-hr-import | cron(0 21 ? * MON-FRI *) | batch-hr-import Lambda |
| expense-org-import | cron(5 21 ? * MON-FRI *) | batch-org-import Lambda |
| expense-journal-export | cron(0 13 ? * MON-FRI *) | batch-journal-export Lambda |

## 2. IAMポリシー設計

### 2.1 Lambda API実行ロール

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RDSConnect",
      "Effect": "Allow",
      "Action": ["rds-db:connect"],
      "Resource": "arn:aws:rds-db:{region}:{account}:dbuser:{resource-id}/expense_app"
    },
    {
      "Sid": "S3Attachments",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::{env}-expense-attachments/*"
    },
    {
      "Sid": "SESSend",
      "Effect": "Allow",
      "Action": ["ses:SendEmail", "ses:SendRawEmail"],
      "Resource": "arn:aws:ses:{region}:{account}:identity/{domain}"
    },
    {
      "Sid": "SecretsRead",
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": [
        "arn:aws:secretsmanager:{region}:{account}:secret:{env}/expense/db-credentials-*",
        "arn:aws:secretsmanager:{region}:{account}:secret:{env}/expense/oidc-config-*"
      ]
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:{region}:{account}:log-group:/aws/lambda/expense-*"
    },
    {
      "Sid": "VPCAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface"
      ],
      "Resource": "*"
    }
  ]
}
```

### 2.2 Lambda バッチ実行ロール

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RDSConnect",
      "Effect": "Allow",
      "Action": ["rds-db:connect"],
      "Resource": "arn:aws:rds-db:{region}:{account}:dbuser:{resource-id}/expense_app"
    },
    {
      "Sid": "S3Batch",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::{env}-expense-batch",
        "arn:aws:s3:::{env}-expense-batch/*"
      ]
    },
    {
      "Sid": "SecretsRead",
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "arn:aws:secretsmanager:{region}:{account}:secret:{env}/expense/db-credentials-*"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:{region}:{account}:log-group:/aws/lambda/expense-batch-*"
    },
    {
      "Sid": "VPCAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface"
      ],
      "Resource": "*"
    }
  ]
}
```

## 3. コスト見積もり（月額概算、production環境）

| サービス | 見積もり | 前提 |
|---------|---------|------|
| Lambda | $15〜30 | 100万リクエスト/月、512MB、平均200ms |
| API Gateway | $10〜20 | 100万リクエスト/月 |
| RDS (db.t3.medium) | $70〜90 | マルチAZ、20GB gp3 |
| S3 | $5〜10 | 50GB保存、10万リクエスト/月 |
| CloudFront | $5〜10 | 50GB転送/月 |
| SES | $1〜5 | 1万通/月 |
| Secrets Manager | $2 | シークレット2つ |
| CloudWatch | $10〜20 | ログ、メトリクス、アラーム |
| NAT Gateway | $70〜100 | 2AZ、データ処理量による |
| WAF | $10〜15 | WebACL + マネージドルール |
| VPCエンドポイント | $30〜40 | Interfaceエンドポイント3つ |
| **合計** | **$230〜340/月** | |

> **注記**: NAT GatewayとVPCエンドポイントがコストの大部分を占める。dev/staging環境ではNAT Gatewayを1AZに削減、VPCエンドポイントを共有することでコスト削減可能。
