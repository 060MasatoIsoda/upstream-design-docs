# 論理コンポーネント構成 — expense-mvp

## 1. AWSサービス構成図

```
+------------------------------------------------------------------+
|                         インターネット                              |
+------------------------------------------------------------------+
                              |
                              v
+------------------------------------------------------------------+
|                      Amazon CloudFront                            |
|                   (フロントエンド配信 + WAF)                       |
+------------------------------------------------------------------+
          |                                    |
          v                                    v
+-------------------+              +------------------------+
| S3 (静的サイト)    |              | Amazon API Gateway     |
| フロントエンド     |              | (REST API)             |
| React SPA         |              | - 認証                  |
+-------------------+              | - スロットリング         |
                                   | - アクセスログ           |
                                   +------------------------+
                                              |
                                              v
                                   +------------------------+
                                   | AWS WAF                |
                                   | - SQLi防御              |
                                   | - レート制限             |
                                   | - 共通攻撃防御           |
                                   +------------------------+
                                              |
                                              v
+------------------------------------------------------------------+
|                          VPC                                      |
|                                                                   |
|  +--------------------+    +--------------------+                 |
|  | パブリックサブネット  |    | パブリックサブネット  |                 |
|  | (AZ-a)             |    | (AZ-c)             |                 |
|  | NAT Gateway        |    | NAT Gateway        |                 |
|  +--------------------+    +--------------------+                 |
|                                                                   |
|  +--------------------+    +--------------------+                 |
|  | プライベートサブネット |    | プライベートサブネット |                 |
|  | (AZ-a)             |    | (AZ-c)             |                 |
|  |                    |    |                    |                  |
|  | Lambda関数群        |    | Lambda関数群        |                 |
|  | - api-handler      |    | - api-handler      |                 |
|  | - batch-hr-import  |    | - batch-hr-import  |                 |
|  | - batch-org-import |    | - batch-org-import |                 |
|  | - batch-journal    |    | - batch-journal    |                 |
|  +--------------------+    +--------------------+                 |
|                                                                   |
|  +--------------------+    +--------------------+                 |
|  | DBサブネット         |    | DBサブネット         |                 |
|  | (AZ-a)             |    | (AZ-c)             |                 |
|  |                    |    |                    |                  |
|  | RDS PostgreSQL     |    | RDS PostgreSQL     |                 |
|  | (プライマリ)        |    | (スタンバイ)        |                 |
|  +--------------------+    +--------------------+                 |
|                                                                   |
+------------------------------------------------------------------+
          |              |              |
          v              v              v
+------------+  +-------------+  +------------------+
| S3          |  | Amazon SES  |  | Secrets Manager  |
| (添付ファイル)|  | (メール通知) |  | (機密情報管理)    |
| (バッチ)    |  |             |  |                  |
+------------+  +-------------+  +------------------+
          |
          v
+------------------------------------------------------------------+
|                      モニタリング・運用                              |
|  +----------------+  +----------------+  +----------------+       |
|  | CloudWatch     |  | CloudWatch     |  | SNS            |       |
|  | Logs           |  | Alarms         |  | (アラーム通知)  |       |
|  +----------------+  +----------------+  +----------------+       |
|  +----------------+                                               |
|  | EventBridge    |                                               |
|  | (バッチスケジュール)|                                             |
|  +----------------+                                               |
+------------------------------------------------------------------+
```

## 2. ネットワーク構成

### 2.1 VPC設計

| 項目 | 設定 |
|------|------|
| VPC CIDR | 10.0.0.0/16 |
| AZ | 2つ（ap-northeast-1a, ap-northeast-1c） |

### 2.2 サブネット設計

| サブネット | CIDR | AZ | 用途 |
|-----------|------|-----|------|
| public-a | 10.0.1.0/24 | AZ-a | NAT Gateway |
| public-c | 10.0.2.0/24 | AZ-c | NAT Gateway |
| private-a | 10.0.11.0/24 | AZ-a | Lambda関数 |
| private-c | 10.0.12.0/24 | AZ-c | Lambda関数 |
| db-a | 10.0.21.0/24 | AZ-a | RDS プライマリ |
| db-c | 10.0.22.0/24 | AZ-c | RDS スタンバイ |

### 2.3 セキュリティグループ

| SG名 | インバウンド | アウトバウンド | 用途 |
|------|------------|-------------|------|
| sg-lambda | なし | 全ポート（VPC内） | Lambda関数 |
| sg-rds | 5432 (sg-lambda) | なし | RDS |

### 2.4 VPCエンドポイント

| エンドポイント | タイプ | 用途 |
|-------------|------|------|
| S3 | Gateway | S3アクセス（無料） |
| Secrets Manager | Interface | 機密情報取得 |
| SES | Interface | メール送信 |
| CloudWatch Logs | Interface | ログ送信 |

## 3. データフロー

### 3.1 API リクエストフロー

```
ブラウザ
  |
  v (HTTPS)
CloudFront --> S3 (静的コンテンツ)
  |
  v (HTTPS, /api/*)
API Gateway
  |
  v (WAFチェック)
WAF
  |
  v (認証チェック)
Lambda (api-handler)
  |
  +---> RDS (データ読み書き、TLS)
  |
  +---> S3 (ファイル操作、HTTPS)
  |
  +---> SES (メール送信、HTTPS)
  |
  +---> Secrets Manager (機密情報取得、HTTPS)
```

### 3.2 バッチ処理フロー

```
EventBridge (スケジュール: 毎日 6:00 JST)
  |
  v
Lambda (batch-hr-import / batch-org-import)
  |
  +---> S3 (CSVファイル取得)
  |
  +---> RDS (マスタデータ更新)
  |
  +---> CloudWatch Logs (実行ログ)

EventBridge (スケジュール: 毎日 22:00 JST)
  |
  v
Lambda (batch-journal-export)
  |
  +---> RDS (承認済みデータ取得)
  |
  +---> S3 (CSVファイル出力)
  |
  +---> CloudWatch Logs (実行ログ)
```

## 4. RDS構成

| 項目 | 設定 |
|------|------|
| エンジン | PostgreSQL 16 |
| インスタンスクラス | db.t3.medium（初期） |
| ストレージ | gp3、20GB（自動拡張有効、最大100GB） |
| マルチAZ | 有効（スタンバイ: AZ-c） |
| 自動バックアップ | 有効（保持期間7日） |
| 暗号化 | 有効（AWS KMS） |
| パラメータグループ | カスタム（ログ設定、接続数調整） |
| 接続数上限 | 100（db.t3.medium デフォルト） |

## 5. S3バケット構成

| バケット名（論理名） | 用途 | 暗号化 | パブリックアクセス | ライフサイクル |
|-------------------|------|--------|-----------------|-------------|
| expense-attachments | 添付ファイル | SSE-S3 | ブロック | 3年後Glacier、4年後削除 |
| expense-batch | バッチ入出力ファイル | SSE-S3 | ブロック | 90日後削除 |
| expense-frontend | フロントエンドSPA | SSE-S3 | CloudFront OAI経由のみ | なし |

## 6. モニタリング・アラーム構成

### 6.1 CloudWatch ダッシュボード

| パネル | メトリクス |
|--------|----------|
| API概要 | リクエスト数、エラー率、レイテンシ（p50/p95/p99） |
| Lambda | 呼出し数、エラー数、Duration、同時実行数 |
| RDS | CPU使用率、接続数、読み書きIOPS、空きストレージ |
| バッチ | 実行回数、成功/失敗、処理時間 |

### 6.2 ログ構成

| ログソース | ロググループ | 保持期間 |
|-----------|------------|---------|
| API Gateway | /aws/apigateway/expense-api | 90日 |
| Lambda (api-handler) | /aws/lambda/expense-api-handler | 90日 |
| Lambda (batch-*) | /aws/lambda/expense-batch-* | 90日 |
| RDS | /aws/rds/expense-db | 90日 |
| アプリケーション | /expense-app/application | 90日 |

## 7. EventBridge スケジュール

| スケジュール名 | cron式 | 対象Lambda | 説明 |
|-------------|--------|-----------|------|
| hr-import-daily | cron(0 21 ? * MON-FRI *) | batch-hr-import | 人事マスタ取込（JST 6:00） |
| org-import-daily | cron(5 21 ? * MON-FRI *) | batch-org-import | 組織マスタ取込（JST 6:05） |
| journal-export-daily | cron(0 13 ? * MON-FRI *) | batch-journal-export | 仕訳出力（JST 22:00） |

## 8. Terraform モジュール構成

```
terraform/
+-- modules/
|   +-- vpc/              # VPC、サブネット、NAT Gateway、VPCエンドポイント
|   +-- rds/              # RDS PostgreSQL、パラメータグループ、SG
|   +-- lambda/           # Lambda関数、レイヤー、IAMロール
|   +-- api-gateway/      # API Gateway、ステージ、使用量プラン
|   +-- s3/               # S3バケット、ライフサイクル、ポリシー
|   +-- cloudfront/       # CloudFront、OAI、WAF関連付け
|   +-- waf/              # WAF WebACL、ルール
|   +-- monitoring/       # CloudWatch、アラーム、SNS、ダッシュボード
|   +-- ses/              # SES、ドメイン検証、送信設定
|   +-- secrets/          # Secrets Manager
|   +-- eventbridge/      # EventBridgeスケジュール
+-- environments/
|   +-- dev/              # 開発環境
|   +-- staging/          # ステージング環境
|   +-- production/       # 本番環境
+-- backend.tf            # Terraformバックエンド（S3 + DynamoDB）
+-- variables.tf          # 共通変数
+-- outputs.tf            # 出力値
```
