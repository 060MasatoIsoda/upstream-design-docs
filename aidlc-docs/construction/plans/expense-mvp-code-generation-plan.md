---
layout: default
title: コード生成計画
---

# コード生成計画 — expense-mvp

## ユニットコンテキスト

- **ユニット名**: expense-mvp
- **プロジェクトタイプ**: グリーンフィールド（単一ユニット）
- **ワークスペースルート**: .
- **コード配置**: ワークスペースルート直下（aidlc-docs/には配置しない）
- **技術スタック**: React 18 + TypeScript 5 + Vite（FE）、Python 3.12 + FastAPI（BE）、PostgreSQL 16（DB）
- **インフラ**: AWS Lambda + API Gateway + RDS + S3 + SES + CloudFront

## プロジェクト構造

```
expense-mvp/                        # ワークスペースルート
+-- backend/                        # バックエンド（Python/FastAPI）
|   +-- app/
|   |   +-- __init__.py
|   |   +-- main.py                 # Lambda ハンドラー + FastAPI アプリ
|   |   +-- config.py               # 設定管理
|   |   +-- dependencies.py         # FastAPI 依存性注入
|   |   +-- models/                 # SQLAlchemy ORM モデル
|   |   |   +-- __init__.py
|   |   |   +-- base.py             # Base, 共通Mixin
|   |   |   +-- user.py             # User, Role, UserRole, Department
|   |   |   +-- expense.py          # Expense, ExpenseItem, ExpenseType, Attachment
|   |   |   +-- approval.py         # ApprovalRoute, ApprovalRecord, DelegationSetting
|   |   |   +-- audit.py            # AuditLog, NotificationLog, BatchExecutionLog
|   |   +-- schemas/                # Pydantic スキーマ
|   |   |   +-- __init__.py
|   |   |   +-- user.py
|   |   |   +-- expense.py
|   |   |   +-- approval.py
|   |   |   +-- dashboard.py
|   |   |   +-- file.py
|   |   |   +-- audit.py
|   |   |   +-- common.py           # ページネーション、共通レスポンス
|   |   +-- repositories/           # リポジトリ層
|   |   |   +-- __init__.py
|   |   |   +-- base.py             # BaseRepository
|   |   |   +-- user.py
|   |   |   +-- expense.py
|   |   |   +-- approval.py
|   |   |   +-- dashboard.py
|   |   |   +-- audit.py
|   |   +-- services/               # サービス層（ビジネスロジック）
|   |   |   +-- __init__.py
|   |   |   +-- auth.py             # AuthService
|   |   |   +-- expense.py          # ExpenseService
|   |   |   +-- workflow.py         # WorkflowService
|   |   |   +-- user.py             # UserService
|   |   |   +-- route.py            # RouteService
|   |   |   +-- dashboard.py        # DashboardService
|   |   |   +-- file.py             # FileService
|   |   |   +-- notification.py     # NotificationService
|   |   |   +-- audit.py            # AuditService
|   |   +-- routers/                # FastAPI ルーター（API層）
|   |   |   +-- __init__.py
|   |   |   +-- auth.py
|   |   |   +-- expenses.py
|   |   |   +-- approvals.py
|   |   |   +-- users.py
|   |   |   +-- approval_routes.py
|   |   |   +-- dashboard.py
|   |   |   +-- files.py
|   |   |   +-- audit_logs.py
|   |   +-- middleware/             # ミドルウェア
|   |   |   +-- __init__.py
|   |   |   +-- auth.py             # OIDC トークン検証
|   |   |   +-- error_handler.py    # グローバルエラーハンドラー
|   |   |   +-- audit.py            # 監査ログミドルウェア
|   |   |   +-- security_headers.py # セキュリティヘッダー
|   |   +-- utils/                  # ユーティリティ
|   |       +-- __init__.py
|   |       +-- logger.py           # 構造化ログ設定
|   |       +-- pagination.py       # ページネーションヘルパー
|   |       +-- exceptions.py       # カスタム例外
|   +-- batch/                      # バッチ処理
|   |   +-- __init__.py
|   |   +-- hr_import.py            # 人事マスタ取込
|   |   +-- org_import.py           # 組織マスタ取込
|   |   +-- journal_export.py       # 会計仕訳出力
|   +-- migrations/                 # Alembic マイグレーション
|   |   +-- env.py
|   |   +-- alembic.ini
|   |   +-- versions/
|   |       +-- 001_initial_schema.py
|   +-- tests/                      # バックエンドテスト
|   |   +-- __init__.py
|   |   +-- conftest.py             # テスト共通設定・フィクスチャ
|   |   +-- factories.py            # テストデータファクトリ
|   |   +-- test_models/
|   |   +-- test_services/
|   |   +-- test_routers/
|   |   +-- test_batch/
|   +-- requirements.txt            # 本番依存
|   +-- requirements-dev.txt        # 開発依存
|   +-- pyproject.toml              # プロジェクト設定
+-- frontend/                       # フロントエンド（React/TypeScript）
|   +-- src/
|   |   +-- main.tsx                # エントリーポイント
|   |   +-- App.tsx                 # ルートコンポーネント
|   |   +-- routes.tsx              # ルーティング定義
|   |   +-- api/                    # API クライアント
|   |   |   +-- client.ts           # Axios インスタンス
|   |   |   +-- expenses.ts
|   |   |   +-- approvals.ts
|   |   |   +-- users.ts
|   |   |   +-- dashboard.ts
|   |   |   +-- files.ts
|   |   |   +-- auth.ts
|   |   +-- components/             # 共通コンポーネント
|   |   |   +-- Layout/
|   |   |   +-- common/
|   |   +-- features/               # 機能別コンポーネント
|   |   |   +-- auth/
|   |   |   +-- expenses/
|   |   |   +-- approvals/
|   |   |   +-- dashboard/
|   |   |   +-- admin/
|   |   |   +-- settings/
|   |   +-- hooks/                  # カスタムフック
|   |   +-- stores/                 # Zustand ストア
|   |   |   +-- authStore.ts
|   |   |   +-- expenseStore.ts
|   |   |   +-- approvalStore.ts
|   |   |   +-- uiStore.ts
|   |   +-- types/                  # TypeScript 型定義
|   |   |   +-- index.ts
|   |   |   +-- expense.ts
|   |   |   +-- user.ts
|   |   |   +-- approval.ts
|   |   +-- utils/                  # ユーティリティ
|   |   +-- i18n/                   # 多言語リソース
|   |       +-- ja.json
|   |       +-- en.json
|   +-- tests/                      # フロントエンドテスト
|   +-- index.html
|   +-- vite.config.ts
|   +-- tsconfig.json
|   +-- package.json
+-- terraform/                      # IaC
|   +-- modules/                    # Terraform モジュール
|   +-- environments/               # 環境別設定
|   +-- backend.tf
|   +-- variables.tf
+-- .github/                        # CI/CD
|   +-- workflows/
|       +-- backend-ci.yml
|       +-- frontend-ci.yml
|       +-- terraform-ci.yml
+-- README.md
```

## コード生成ステップ

### Step 1: プロジェクト構造セットアップ
- [ ] バックエンドプロジェクト構造の作成（ディレクトリ、__init__.py）
- [ ] pyproject.toml、requirements.txt、requirements-dev.txt の作成
- [ ] フロントエンドプロジェクト構造の作成
- [ ] package.json、vite.config.ts、tsconfig.json の作成
- [ ] README.md の作成

### Step 2: バックエンド共通基盤
- [ ] app/config.py — 環境変数・設定管理
- [ ] app/utils/logger.py — 構造化ログ設定（structlog + aws-lambda-powertools）
- [ ] app/utils/exceptions.py — カスタム例外クラス
- [ ] app/utils/pagination.py — ページネーションヘルパー
- [ ] app/middleware/error_handler.py — グローバルエラーハンドラー
- [ ] app/middleware/security_headers.py — セキュリティヘッダーミドルウェア
- [ ] app/middleware/auth.py — OIDC トークン検証ミドルウェア
- [ ] app/middleware/audit.py — 監査ログミドルウェア

### Step 3: データベースモデル（SQLAlchemy ORM）
- [ ] app/models/base.py — Base クラス、共通 Mixin（TimestampMixin, UUIDMixin）
- [ ] app/models/user.py — User, Role, UserRole, Department モデル
- [ ] app/models/expense.py — Expense, ExpenseItem, ExpenseType, Attachment モデル
- [ ] app/models/approval.py — ApprovalRoute, ApprovalRecord, DelegationSetting モデル
- [ ] app/models/audit.py — AuditLog, NotificationLog, BatchExecutionLog モデル
- [ ] app/models/__init__.py — 全モデルのエクスポート

### Step 4: データベースモデル ユニットテスト
- [ ] tests/conftest.py — テスト共通設定（テストDB、セッション、フィクスチャ）
- [ ] tests/factories.py — factory-boy テストデータファクトリ
- [ ] tests/test_models/ — モデルのバリデーション・リレーションテスト

### Step 5: データベースモデル サマリー
- [ ] aidlc-docs/construction/expense-mvp/code/models-summary.md

### Step 6: Pydantic スキーマ
- [ ] app/schemas/common.py — ページネーション、共通レスポンス
- [ ] app/schemas/user.py — ユーザー関連スキーマ
- [ ] app/schemas/expense.py — 経費精算関連スキーマ
- [ ] app/schemas/approval.py — 承認関連スキーマ
- [ ] app/schemas/dashboard.py — ダッシュボード関連スキーマ
- [ ] app/schemas/file.py — ファイル関連スキーマ
- [ ] app/schemas/audit.py — 監査ログ関連スキーマ

### Step 7: リポジトリ層
- [ ] app/repositories/base.py — BaseRepository（CRUD共通操作）
- [ ] app/repositories/user.py — UserRepository, DepartmentRepository
- [ ] app/repositories/expense.py — ExpenseRepository, ExpenseItemRepository
- [ ] app/repositories/approval.py — ApprovalRouteRepository, ApprovalRecordRepository, DelegationRepository
- [ ] app/repositories/dashboard.py — DashboardRepository（集計クエリ）
- [ ] app/repositories/audit.py — AuditLogRepository

### Step 8: リポジトリ層 ユニットテスト
- [ ] tests/test_repositories/ — リポジトリ層のCRUD・クエリテスト

### Step 9: リポジトリ層 サマリー
- [ ] aidlc-docs/construction/expense-mvp/code/repositories-summary.md

### Step 10: サービス層（ビジネスロジック）
- [ ] app/services/auth.py — AuthService（OIDC認証フロー、トークン管理）
- [ ] app/services/expense.py — ExpenseService（経費精算CRUD、ステータス遷移）
- [ ] app/services/workflow.py — WorkflowService（承認ルート判定、承認・差戻し処理）
- [ ] app/services/user.py — UserService（ユーザーCRUD、ロール管理）
- [ ] app/services/route.py — RouteService（承認ルート設定・管理）
- [ ] app/services/dashboard.py — DashboardService（集計データ生成）
- [ ] app/services/file.py — FileService（S3ファイル操作）
- [ ] app/services/notification.py — NotificationService（SESメール送信）
- [ ] app/services/audit.py — AuditService（監査ログ記録）

### Step 11: サービス層 ユニットテスト
- [ ] tests/test_services/ — サービス層のビジネスロジックテスト

### Step 12: サービス層 サマリー
- [ ] aidlc-docs/construction/expense-mvp/code/services-summary.md

### Step 13: API層（FastAPI ルーター）
- [ ] app/dependencies.py — FastAPI 依存性注入（DB セッション、認証ユーザー）
- [ ] app/routers/auth.py — 認証API（/api/v1/auth/*）
- [ ] app/routers/expenses.py — 経費精算API（/api/v1/expenses/*）
- [ ] app/routers/approvals.py — 承認API（/api/v1/approvals/*）
- [ ] app/routers/users.py — ユーザー管理API（/api/v1/users/*）
- [ ] app/routers/approval_routes.py — 承認ルート管理API（/api/v1/approval-routes/*）
- [ ] app/routers/dashboard.py — ダッシュボードAPI（/api/v1/dashboard/*）
- [ ] app/routers/files.py — ファイル管理API（/api/v1/files/*）
- [ ] app/routers/audit_logs.py — 監査ログAPI（/api/v1/audit-logs/*）
- [ ] app/main.py — FastAPI アプリ + Lambda ハンドラー

### Step 14: API層 ユニットテスト
- [ ] tests/test_routers/ — API層のエンドポイントテスト

### Step 15: API層 サマリー
- [ ] aidlc-docs/construction/expense-mvp/code/api-summary.md

### Step 16: バッチ処理
- [ ] batch/hr_import.py — 人事マスタ取込 Lambda ハンドラー
- [ ] batch/org_import.py — 組織マスタ取込 Lambda ハンドラー
- [ ] batch/journal_export.py — 会計仕訳出力 Lambda ハンドラー

### Step 17: バッチ処理 ユニットテスト
- [ ] tests/test_batch/ — バッチ処理テスト

### Step 18: バッチ処理 サマリー
- [ ] aidlc-docs/construction/expense-mvp/code/batch-summary.md

### Step 19: データベースマイグレーション
- [ ] migrations/alembic.ini — Alembic 設定
- [ ] migrations/env.py — マイグレーション環境設定
- [ ] migrations/versions/001_initial_schema.py — 初期スキーマ

### Step 20: フロントエンド共通基盤
- [ ] src/main.tsx — エントリーポイント
- [ ] src/App.tsx — ルートコンポーネント
- [ ] src/routes.tsx — ルーティング定義
- [ ] src/api/client.ts — Axios インスタンス（認証トークン付与、エラーハンドリング）
- [ ] src/types/ — TypeScript 型定義
- [ ] src/stores/ — Zustand ストア（auth, expense, approval, ui）
- [ ] src/i18n/ — 多言語リソース（ja.json, en.json）
- [ ] src/hooks/ — カスタムフック

### Step 21: フロントエンド共通コンポーネント
- [ ] src/components/Layout/ — レイアウト（Header, Sidebar, Footer）
- [ ] src/components/common/ — 共通UI部品（LoadingSpinner, ErrorBoundary, Toast）

### Step 22: フロントエンド API クライアント
- [ ] src/api/auth.ts — 認証API
- [ ] src/api/expenses.ts — 経費精算API
- [ ] src/api/approvals.ts — 承認API
- [ ] src/api/users.ts — ユーザー管理API
- [ ] src/api/dashboard.ts — ダッシュボードAPI
- [ ] src/api/files.ts — ファイル管理API

### Step 23: フロントエンド機能画面 — 認証・ホーム
- [ ] src/features/auth/ — ログイン画面、認証コールバック
- [ ] src/features/expenses/ExpenseListPage.tsx — 申請一覧（SCR-003）
- [ ] ホーム画面（SCR-002）

### Step 24: フロントエンド機能画面 — 経費精算
- [ ] src/features/expenses/ExpenseCreatePage.tsx — 申請作成（SCR-004）
- [ ] src/features/expenses/ExpenseEditPage.tsx — 申請編集（SCR-005）
- [ ] src/features/expenses/ExpenseDetailPage.tsx — 申請詳細（SCR-006）
- [ ] src/features/expenses/components/ — フォーム部品（ExpenseForm, ExpenseItemRow, AttachmentSection）

### Step 25: フロントエンド機能画面 — 承認
- [ ] src/features/approvals/ApprovalListPage.tsx — 承認待ち一覧（SCR-007）
- [ ] src/features/approvals/ApprovalPage.tsx — 承認画面（SCR-008）

### Step 26: フロントエンド機能画面 — ダッシュボード・管理
- [ ] src/features/dashboard/DashboardPage.tsx — ダッシュボード（SCR-009）
- [ ] src/features/admin/UserManagementPage.tsx — ユーザー管理（SCR-010）
- [ ] src/features/admin/RouteManagementPage.tsx — 承認ルート管理（SCR-011）
- [ ] src/features/admin/DelegationManagementPage.tsx — 代理設定管理（SCR-012）
- [ ] src/features/admin/AuditLogPage.tsx — 監査ログ（SCR-013）
- [ ] src/features/settings/SettingsPage.tsx — 設定画面（SCR-014）

### Step 27: フロントエンド ユニットテスト
- [ ] tests/ — 主要コンポーネントのユニットテスト（Vitest + React Testing Library）

### Step 28: フロントエンド サマリー
- [ ] aidlc-docs/construction/expense-mvp/code/frontend-summary.md

### Step 29: Terraform インフラコード
- [ ] terraform/backend.tf — Terraform バックエンド設定
- [ ] terraform/variables.tf — 共通変数定義
- [ ] terraform/modules/vpc/ — VPC、サブネット、NAT Gateway
- [ ] terraform/modules/rds/ — RDS PostgreSQL
- [ ] terraform/modules/lambda/ — Lambda 関数、レイヤー、IAM ロール
- [ ] terraform/modules/api-gateway/ — API Gateway
- [ ] terraform/modules/s3/ — S3 バケット
- [ ] terraform/modules/cloudfront/ — CloudFront + WAF
- [ ] terraform/modules/monitoring/ — CloudWatch、アラーム、SNS
- [ ] terraform/modules/ses/ — SES 設定
- [ ] terraform/modules/secrets/ — Secrets Manager
- [ ] terraform/modules/eventbridge/ — EventBridge スケジュール
- [ ] terraform/environments/ — 環境別 tfvars

### Step 30: CI/CD パイプライン
- [ ] .github/workflows/backend-ci.yml — バックエンド CI
- [ ] .github/workflows/frontend-ci.yml — フロントエンド CI
- [ ] .github/workflows/terraform-ci.yml — Terraform CI

### Step 31: デプロイ成果物 サマリー
- [ ] aidlc-docs/construction/expense-mvp/code/deployment-summary.md

## 備考
- 全31ステップ、推定ファイル数: 約120ファイル
- バックエンド → フロントエンド → インフラ → CI/CD の順で生成
- 各層のテストは実装直後に生成（テスト実行はビルドとテストステージで実施）
- セキュリティ拡張ルール（SECURITY-01〜15）は各ステップで準拠を確認
