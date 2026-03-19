---
layout: default
title: 技術スタック決定
---

# 技術スタック決定 — expense-mvp

## 1. 技術スタック全体像

```
+-----------------------------------------------------------+
|                    フロントエンド                            |
|  React 18 + TypeScript 5 + Vite                           |
|  MUI v5 | Zustand | React Router | React Query            |
|  Vitest + React Testing Library                            |
+-----------------------------------------------------------+
                          |
                     HTTPS (REST)
                          |
+-----------------------------------------------------------+
|                    AWS インフラ                              |
|  API Gateway (REST) --> Lambda (Python 3.12)               |
|  RDS PostgreSQL 16 | S3 | SES | Secrets Manager           |
|  CloudWatch | SNS | EventBridge                            |
+-----------------------------------------------------------+
                          |
+-----------------------------------------------------------+
|                    IaC / CI/CD                              |
|  Terraform | GitHub Actions                                |
+-----------------------------------------------------------+
```

## 2. バックエンド技術スタック

### 2.1 ランタイム・フレームワーク

| ライブラリ | バージョン | 用途 |
|-----------|----------|------|
| Python | 3.12 | ランタイム |
| FastAPI | 0.115+ | Webフレームワーク |
| Uvicorn | 0.30+ | ASGIサーバー（ローカル開発用） |
| Pydantic | 2.x | データバリデーション・シリアライゼーション |

### 2.2 データベース・ORM

| ライブラリ | バージョン | 用途 |
|-----------|----------|------|
| SQLAlchemy | 2.x | ORM |
| Alembic | 1.13+ | DBマイグレーション |
| psycopg2-binary | 2.9+ | PostgreSQLドライバー |

### 2.3 認証・セキュリティ

| ライブラリ | バージョン | 用途 |
|-----------|----------|------|
| python-jose | 3.3+ | JWT検証（JWKS） |
| cryptography | 42+ | 暗号化ユーティリティ |

### 2.4 AWS連携

| ライブラリ | バージョン | 用途 |
|-----------|----------|------|
| boto3 | 1.34+ | AWS SDK（S3, SES, Secrets Manager） |

### 2.5 ログ・ユーティリティ

| ライブラリ | バージョン | 用途 |
|-----------|----------|------|
| structlog | 24+ | 構造化ログ |
| aws-lambda-powertools | 2.x | Lambda向けユーティリティ（ログ、トレース、メトリクス） |

### 2.6 テスト

| ライブラリ | バージョン | 用途 |
|-----------|----------|------|
| pytest | 8+ | テストフレームワーク |
| pytest-cov | 5+ | カバレッジ計測 |
| httpx | 0.27+ | FastAPIテストクライアント |
| moto | 5+ | AWSサービスモック |
| factory-boy | 3.3+ | テストデータファクトリ |

### 2.7 コード品質

| ツール | バージョン | 用途 |
|--------|----------|------|
| Ruff | 0.6+ | リンター + フォーマッター |
| mypy | 1.11+ | 型チェック |

## 3. フロントエンド技術スタック

### 3.1 コア

| ライブラリ | バージョン | 用途 |
|-----------|----------|------|
| React | 18.x | UIライブラリ |
| TypeScript | 5.x | 型安全な開発 |
| Vite | 5.x | ビルドツール |

### 3.2 UI・状態管理

| ライブラリ | バージョン | 用途 |
|-----------|----------|------|
| MUI (Material UI) | 5.x | UIコンポーネントライブラリ |
| @mui/x-data-grid | 7.x | データテーブル |
| @mui/x-date-pickers | 7.x | 日付ピッカー |
| Zustand | 4.x | 状態管理 |
| React Router | 6.x | ルーティング |
| TanStack Query (React Query) | 5.x | サーバー状態管理・キャッシュ |

### 3.3 フォーム・バリデーション

| ライブラリ | バージョン | 用途 |
|-----------|----------|------|
| React Hook Form | 7.x | フォーム管理 |
| Zod | 3.x | スキーマバリデーション |

### 3.4 国際化

| ライブラリ | バージョン | 用途 |
|-----------|----------|------|
| react-i18next | 14.x | 多言語対応（日本語・英語） |
| i18next | 23.x | 国際化フレームワーク |

### 3.5 グラフ・可視化

| ライブラリ | バージョン | 用途 |
|-----------|----------|------|
| Recharts | 2.x | ダッシュボードグラフ |

### 3.6 テスト

| ライブラリ | バージョン | 用途 |
|-----------|----------|------|
| Vitest | 2.x | テストフレームワーク |
| @testing-library/react | 16.x | Reactコンポーネントテスト |
| @testing-library/user-event | 14.x | ユーザーイベントシミュレーション |
| MSW (Mock Service Worker) | 2.x | APIモック |

### 3.7 コード品質

| ツール | バージョン | 用途 |
|--------|----------|------|
| ESLint | 9.x | リンター |
| Prettier | 3.x | フォーマッター |

## 4. インフラ・運用

### 4.1 AWSサービス

| サービス | 用途 |
|---------|------|
| API Gateway (REST API) | APIエンドポイント、認証、スロットリング |
| Lambda (Python 3.12) | バックエンドAPI実行 |
| RDS for PostgreSQL 16 | データベース |
| S3 | 添付ファイル・バッチファイル保存 |
| SES | メール通知送信 |
| Secrets Manager | DB接続情報・IdP設定の管理 |
| CloudWatch | ログ・メトリクス・アラーム |
| SNS | アラーム通知 |
| EventBridge | バッチスケジュール実行 |
| CloudFront | フロントエンド配信（S3オリジン） |
| WAF | API Gateway保護 |

### 4.2 IaC

| ツール | バージョン | 用途 |
|--------|----------|------|
| Terraform | 1.9+ | インフラ定義・管理 |
| AWS Provider | 5.x | AWSリソースプロバイダー |

### 4.3 CI/CD

| ツール | 用途 |
|--------|------|
| GitHub Actions | CI/CDパイプライン |
| GitHub Dependabot | 依存脆弱性スキャン |

### 4.4 CI/CDパイプライン概要

```
[Push/PR] --> [Lint + Type Check] --> [Unit Test] --> [Build]
                                                        |
                                              [Integration Test]
                                                        |
                                              [Deploy to Staging]
                                                        |
                                              [E2E Test (将来)]
                                                        |
                                              [Deploy to Production]
```

## 5. Lambda実行環境の詳細

### 5.1 構成

| 項目 | 設定 |
|------|------|
| ランタイム | Python 3.12 |
| ハンドラー | Lambda ネイティブハンドラー |
| メモリ | 512MB（初期値、チューニング対象） |
| タイムアウト | API: 30秒 / バッチ: 900秒（15分） |
| 予約済み同時実行 | 50 |
| レイヤー | 共通依存ライブラリ用Lambdaレイヤー |

### 5.2 Lambda関数構成

| 関数名 | 用途 | トリガー |
|--------|------|---------|
| api-handler | REST API処理 | API Gateway |
| batch-hr-import | 人事マスタ取込 | EventBridge（日次） |
| batch-org-import | 組織マスタ取込 | EventBridge（日次） |
| batch-journal-export | 会計仕訳出力 | EventBridge（日次） |

## 6. 技術選定の理由

| 決定事項 | 選択 | 理由 |
|---------|------|------|
| SQLAlchemy 2.x | 最新版 | async対応強化、型ヒント改善、長期サポート |
| Lambda直接実行 | Pythonランタイム | シンプル、コスト効率、MVPに適切 |
| pytest + Vitest | テストFW | Pythonエコシステム標準 + Viteとの親和性 |
| GitHub Actions | CI/CD | GitHub連携、豊富なアクション、コスト効率 |
| Terraform | IaC | マルチクラウド対応、宣言的、豊富なプロバイダー |
| RDS for PostgreSQL | DB | マネージド、安定、コスト予測可能 |
