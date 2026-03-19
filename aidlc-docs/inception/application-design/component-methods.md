# コンポーネントメソッド定義

> **注記**: 本ドキュメントではメソッドシグネチャと高レベルの目的を定義する。
> 詳細なビジネスルールはCONSTRUCTIONフェーズの機能設計で定義する。

## 1. BE-EXPENSE: 経費精算API

| メソッド | HTTPメソッド | パス | 入力 | 出力 | 目的 |
|---------|------------|------|------|------|------|
| create_expense | POST | /api/v1/expenses | ExpenseCreateRequest | ExpenseResponse | 経費精算申請の新規作成 |
| update_expense | PUT | /api/v1/expenses/{id} | ExpenseUpdateRequest | ExpenseResponse | 経費精算申請の更新 |
| get_expense | GET | /api/v1/expenses/{id} | - | ExpenseDetailResponse | 経費精算申請の詳細取得 |
| list_expenses | GET | /api/v1/expenses | ExpenseSearchParams | PaginatedExpenseList | 経費精算申請の一覧取得（検索・フィルタ） |
| submit_expense | POST | /api/v1/expenses/{id}/submit | - | ExpenseResponse | 経費精算申請の提出 |
| cancel_expense | POST | /api/v1/expenses/{id}/cancel | - | ExpenseResponse | 経費精算申請の取消し |
| save_draft | POST | /api/v1/expenses/{id}/draft | ExpenseUpdateRequest | ExpenseResponse | 下書き保存 |

## 2. BE-WORKFLOW: 承認ワークフローAPI

| メソッド | HTTPメソッド | パス | 入力 | 出力 | 目的 |
|---------|------------|------|------|------|------|
| approve | POST | /api/v1/approvals/{id}/approve | ApprovalActionRequest | ApprovalResponse | 申請の承認 |
| reject | POST | /api/v1/approvals/{id}/reject | ApprovalActionRequest | ApprovalResponse | 申請の差戻し |
| list_pending | GET | /api/v1/approvals/pending | PaginationParams | PaginatedApprovalList | 承認待ち一覧の取得 |
| get_approval_history | GET | /api/v1/approvals/{expense_id}/history | - | ApprovalHistoryResponse | 承認履歴の取得 |
| get_approval_route | GET | /api/v1/approvals/{expense_id}/route | - | ApprovalRouteResponse | 承認ルートの取得 |

## 3. BE-USER: ユーザー管理API

| メソッド | HTTPメソッド | パス | 入力 | 出力 | 目的 |
|---------|------------|------|------|------|------|
| create_user | POST | /api/v1/users | UserCreateRequest | UserResponse | ユーザーの新規登録 |
| update_user | PUT | /api/v1/users/{id} | UserUpdateRequest | UserResponse | ユーザー情報の更新 |
| get_user | GET | /api/v1/users/{id} | - | UserDetailResponse | ユーザー詳細の取得 |
| list_users | GET | /api/v1/users | UserSearchParams | PaginatedUserList | ユーザー一覧の取得 |
| deactivate_user | POST | /api/v1/users/{id}/deactivate | - | UserResponse | ユーザーの無効化 |
| assign_role | POST | /api/v1/users/{id}/roles | RoleAssignRequest | UserResponse | ロールの割り当て |
| set_delegate | POST | /api/v1/users/{id}/delegate | DelegateRequest | DelegateResponse | 代理設定 |
| get_me | GET | /api/v1/users/me | - | UserDetailResponse | ログインユーザー情報の取得 |

## 4. BE-ROUTE: 承認ルート管理API

| メソッド | HTTPメソッド | パス | 入力 | 出力 | 目的 |
|---------|------------|------|------|------|------|
| create_route | POST | /api/v1/approval-routes | RouteCreateRequest | RouteResponse | 承認ルートの作成 |
| update_route | PUT | /api/v1/approval-routes/{id} | RouteUpdateRequest | RouteResponse | 承認ルートの更新 |
| get_route | GET | /api/v1/approval-routes/{id} | - | RouteDetailResponse | 承認ルートの詳細取得 |
| list_routes | GET | /api/v1/approval-routes | RouteSearchParams | PaginatedRouteList | 承認ルート一覧の取得 |
| delete_route | DELETE | /api/v1/approval-routes/{id} | - | - | 承認ルートの削除 |

## 5. BE-DASHBOARD: ダッシュボードAPI

| メソッド | HTTPメソッド | パス | 入力 | 出力 | 目的 |
|---------|------------|------|------|------|------|
| get_summary | GET | /api/v1/dashboard/summary | DashboardParams | DashboardSummary | 申請件数・処理状況のサマリー取得 |
| get_department_stats | GET | /api/v1/dashboard/departments | DashboardParams | DepartmentStatsResponse | 部門別集計の取得 |
| get_stale_applications | GET | /api/v1/dashboard/stale | PaginationParams | PaginatedExpenseList | 滞留申請一覧の取得 |

## 6. BE-NOTIFICATION: 通知サービス

| メソッド | 種別 | 入力 | 出力 | 目的 |
|---------|------|------|------|------|
| send_approval_request | 内部 | NotificationRequest | NotificationResult | 承認依頼メール送信 |
| send_approval_complete | 内部 | NotificationRequest | NotificationResult | 承認完了メール送信 |
| send_rejection_notice | 内部 | NotificationRequest | NotificationResult | 差戻しメール送信 |

## 7. BE-FILE: ファイル管理API

| メソッド | HTTPメソッド | パス | 入力 | 出力 | 目的 |
|---------|------------|------|------|------|------|
| upload_file | POST | /api/v1/files/upload | MultipartFile | FileUploadResponse | ファイルアップロード |
| download_file | GET | /api/v1/files/{id}/download | - | 署名付きURL（リダイレクト） | ファイルダウンロード |
| get_file_metadata | GET | /api/v1/files/{id} | - | FileMetadataResponse | ファイルメタデータ取得 |
| delete_file | DELETE | /api/v1/files/{id} | - | - | ファイル削除 |

## 8. BE-AUDIT: 監査ログAPI

| メソッド | HTTPメソッド | パス | 入力 | 出力 | 目的 |
|---------|------------|------|------|------|------|
| list_audit_logs | GET | /api/v1/audit-logs | AuditLogSearchParams | PaginatedAuditLogList | 監査ログ一覧の取得 |
| get_audit_log | GET | /api/v1/audit-logs/{id} | - | AuditLogDetailResponse | 監査ログ詳細の取得 |
| record_audit_log | 内部 | AuditLogEntry | - | 監査ログの記録（内部呼出し） |

## 9. BE-BATCH: バッチ処理

| メソッド | 種別 | 入力 | 出力 | 目的 |
|---------|------|------|------|------|
| import_hr_master | バッチ | CSVファイル | BatchResult | 人事マスタ取込 |
| import_org_master | バッチ | CSVファイル | BatchResult | 組織マスタ取込 |
| export_journal_entries | バッチ | ExportParams | CSVファイル | 会計仕訳データ出力 |

## 10. BE-AUTH: 認証・認可

| メソッド | HTTPメソッド | パス | 入力 | 出力 | 目的 |
|---------|------------|------|------|------|------|
| login_callback | GET | /api/v1/auth/callback | OIDC認可コード | TokenResponse | OIDC認証コールバック |
| refresh_token | POST | /api/v1/auth/refresh | RefreshTokenRequest | TokenResponse | トークンリフレッシュ |
| logout | POST | /api/v1/auth/logout | - | - | ログアウト |
