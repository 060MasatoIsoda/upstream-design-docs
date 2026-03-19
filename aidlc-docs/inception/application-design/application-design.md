---
layout: default
title: アプリケーション設計書（統合版）
---

# アプリケーション設計書（統合版）

## 1. 設計概要

### 1.1 対象システム
統合業務申請・承認プラットフォーム MVP（経費精算モジュール）

### 1.2 技術スタック

| レイヤー | 技術 | 備考 |
|---------|------|------|
| フロントエンド | React + TypeScript | SPA |
| 状態管理 | Zustand | 軽量状態管理 |
| UIライブラリ | MUI（Material UI） | Material Design準拠 |
| バックエンド | Python + FastAPI | REST API |
| ORM | SQLAlchemy | データアクセス |
| DBマイグレーション | Alembic | スキーマ管理 |
| データベース | PostgreSQL | RDS |
| インフラ | AWS サーバーレス | Lambda + API Gateway |
| ファイルストレージ | Amazon S3 | 添付ファイル保存 |
| メール送信 | Amazon SES | 通知メール |
| 認証 | SAML / OIDC | 外部IdP連携 |

### 1.3 アーキテクチャパターン
レイヤードアーキテクチャ（3層構造: Router → Service → Repository）

## 2. システム構成図

```
+---------------------------------------------------------------+
|                    クライアント（ブラウザ）                       |
|  +----------+ +----------+ +----------+ +----------+          |
|  | FE-AUTH  | |FE-EXPENSE| |FE-APPROVE| |FE-INQUIRY|          |
|  +----------+ +----------+ +----------+ +----------+          |
|  +----------+ +----------+ +----------+                       |
|  |FE-DASHBD | | FE-ADMIN | |FE-COMMON |                       |
|  +----------+ +----------+ +----------+                       |
+---------------------------------------------------------------+
                            |
                       HTTPS (REST)
                            |
+---------------------------------------------------------------+
|                   Amazon API Gateway                           |
|                   (認証・スロットリング)                         |
+---------------------------------------------------------------+
                            |
                      Lambda関数
                            |
+---------------------------------------------------------------+
|                   FastAPI アプリケーション                       |
|  +----------+ +----------+ +----------+ +----------+          |
|  | BE-AUTH  | |BE-EXPENSE| |BE-WORKFLW| | BE-USER  |          |
|  +----------+ +----------+ +----------+ +----------+          |
|  +----------+ +----------+ +----------+ +----------+          |
|  | BE-ROUTE | |BE-DASHBD | | BE-FILE  | |BE-NOTIFY |          |
|  +----------+ +----------+ +----------+ +----------+          |
|  +----------+ +----------+                                    |
|  | BE-AUDIT | | BE-BATCH |                                    |
|  +----------+ +----------+                                    |
+---------------------------------------------------------------+
         |              |              |              |
    +--------+    +---------+    +---------+    +---------+
    |PostgreSQL|   |Amazon S3|   |Amazon SES|   |外部 IdP |
    | (RDS)   |   |(ファイル)|   | (メール) |   |(認証)   |
    +--------+    +---------+    +---------+    +---------+
```

## 3. コンポーネント一覧

### 3.1 フロントエンド層（7コンポーネント）

| ID | コンポーネント名 | 目的 |
|----|----------------|------|
| FE-AUTH | 認証コンポーネント | OIDC認証フロー、トークン管理 |
| FE-EXPENSE | 経費精算画面 | 申請の起票・編集・表示 |
| FE-APPROVAL | 承認画面 | 承認待ち一覧、承認・差戻し |
| FE-INQUIRY | 照会画面 | 申請状況の検索・一覧 |
| FE-DASHBOARD | ダッシュボード画面 | 集計・統計情報の表示 |
| FE-ADMIN | 管理画面 | ユーザー・ルート・監査ログ管理 |
| FE-COMMON | 共通UIコンポーネント | レイアウト・通知・多言語 |

### 3.2 バックエンド層（10サービス）

| ID | サービス名 | 目的 |
|----|-----------|------|
| BE-AUTH | 認証サービス | OIDC認証検証、RBAC |
| BE-EXPENSE | 経費精算サービス | 経費精算CRUD、ビジネスロジック |
| BE-WORKFLOW | 承認ワークフローサービス | 承認ルート判定、承認プロセス管理 |
| BE-USER | ユーザー管理サービス | ユーザー・ロール・組織管理 |
| BE-ROUTE | 承認ルート管理サービス | 承認ルート設定・管理 |
| BE-DASHBOARD | ダッシュボードサービス | 集計データ生成 |
| BE-NOTIFICATION | 通知サービス | メール通知送信 |
| BE-FILE | ファイル管理サービス | 添付ファイル管理 |
| BE-AUDIT | 監査ログサービス | 操作ログ記録・検索 |
| BE-BATCH | バッチ処理サービス | 外部システム連携 |

### 3.3 データアクセス層

| ID | コンポーネント名 | 目的 |
|----|----------------|------|
| DA-REPO | リポジトリ層 | SQLAlchemy ORMによるDB操作 |
| DA-MIGRATION | マイグレーション管理 | Alembicによるスキーマ管理 |

## 4. API エンドポイント一覧

### 4.1 認証API
| メソッド | パス | 目的 |
|---------|------|------|
| GET | /api/v1/auth/callback | OIDC認証コールバック |
| POST | /api/v1/auth/refresh | トークンリフレッシュ |
| POST | /api/v1/auth/logout | ログアウト |

### 4.2 経費精算API
| メソッド | パス | 目的 |
|---------|------|------|
| POST | /api/v1/expenses | 申請作成 |
| PUT | /api/v1/expenses/{id} | 申請更新 |
| GET | /api/v1/expenses/{id} | 申請詳細取得 |
| GET | /api/v1/expenses | 申請一覧取得 |
| POST | /api/v1/expenses/{id}/submit | 申請提出 |
| POST | /api/v1/expenses/{id}/cancel | 申請取消し |
| POST | /api/v1/expenses/{id}/draft | 下書き保存 |

### 4.3 承認ワークフローAPI
| メソッド | パス | 目的 |
|---------|------|------|
| POST | /api/v1/approvals/{id}/approve | 承認 |
| POST | /api/v1/approvals/{id}/reject | 差戻し |
| GET | /api/v1/approvals/pending | 承認待ち一覧 |
| GET | /api/v1/approvals/{expense_id}/history | 承認履歴 |
| GET | /api/v1/approvals/{expense_id}/route | 承認ルート |

### 4.4 ユーザー管理API
| メソッド | パス | 目的 |
|---------|------|------|
| POST | /api/v1/users | ユーザー登録 |
| PUT | /api/v1/users/{id} | ユーザー更新 |
| GET | /api/v1/users/{id} | ユーザー詳細 |
| GET | /api/v1/users | ユーザー一覧 |
| POST | /api/v1/users/{id}/deactivate | ユーザー無効化 |
| POST | /api/v1/users/{id}/roles | ロール割当 |
| POST | /api/v1/users/{id}/delegate | 代理設定 |
| GET | /api/v1/users/me | ログインユーザー情報 |

### 4.5 承認ルート管理API
| メソッド | パス | 目的 |
|---------|------|------|
| POST | /api/v1/approval-routes | ルート作成 |
| PUT | /api/v1/approval-routes/{id} | ルート更新 |
| GET | /api/v1/approval-routes/{id} | ルート詳細 |
| GET | /api/v1/approval-routes | ルート一覧 |
| DELETE | /api/v1/approval-routes/{id} | ルート削除 |

### 4.6 ダッシュボードAPI
| メソッド | パス | 目的 |
|---------|------|------|
| GET | /api/v1/dashboard/summary | サマリー |
| GET | /api/v1/dashboard/departments | 部門別集計 |
| GET | /api/v1/dashboard/stale | 滞留申請一覧 |

### 4.7 ファイル管理API
| メソッド | パス | 目的 |
|---------|------|------|
| POST | /api/v1/files/upload | アップロード |
| GET | /api/v1/files/{id}/download | ダウンロード |
| GET | /api/v1/files/{id} | メタデータ取得 |
| DELETE | /api/v1/files/{id} | ファイル削除 |

### 4.8 監査ログAPI
| メソッド | パス | 目的 |
|---------|------|------|
| GET | /api/v1/audit-logs | 監査ログ一覧 |
| GET | /api/v1/audit-logs/{id} | 監査ログ詳細 |

## 5. サービス間依存関係

### 主要な依存関係

| 呼出し元 | 呼出し先 | 目的 |
|---------|---------|------|
| ExpenseService | WorkflowService | 承認ルート生成 |
| ExpenseService | FileService | 添付ファイル関連付け |
| WorkflowService | UserService | 承認者情報取得 |
| WorkflowService | NotificationService | 通知送信 |
| WorkflowService | RouteService | ルート判定 |
| RouteService | UserService | 上長・部門長取得 |
| AuthService | UserService | ユーザー情報取得/作成 |
| 全サービス | AuditService | 操作ログ記録 |

### 外部サービス依存

| サービス | 外部サービス | 目的 |
|---------|------------|------|
| FileService | Amazon S3 | ファイル保存・取得 |
| NotificationService | Amazon SES | メール送信 |
| AuthService | 外部IdP | OIDC認証 |
| BatchService | Amazon S3 | バッチファイル入出力 |

## 6. 設計上の決定事項

| 決定事項 | 選択 | 理由 |
|---------|------|------|
| 状態管理 | Zustand | 軽量でシンプル、MVPに適切 |
| APIスタイル | REST API（OpenAPI準拠） | 標準的、ツールサポート充実 |
| アーキテクチャ | レイヤード（3層） | シンプルで理解しやすい、MVPに適切 |
| DBマイグレーション | Alembic | SQLAlchemy連携、Pythonエコシステム標準 |
| UIライブラリ | MUI | 豊富なコンポーネント、業務システムに適切 |
