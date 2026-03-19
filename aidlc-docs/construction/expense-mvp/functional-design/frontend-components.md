---
layout: default
title: フロントエンドコンポーネント設計
---

# フロントエンドコンポーネント設計 — expense-mvp

## 1. 画面一覧

| 画面ID | 画面名 | パス | 説明 | 必要ロール |
|--------|--------|------|------|-----------|
| SCR-001 | ログイン画面 | /login | IdPへのリダイレクト | 全員 |
| SCR-002 | ホーム画面 | / | 申請サマリー・承認待ち件数表示 | 全員 |
| SCR-003 | 経費精算申請一覧 | /expenses | 自分の申請一覧 | 全員 |
| SCR-004 | 経費精算申請作成 | /expenses/new | 新規申請フォーム | 全員 |
| SCR-005 | 経費精算申請編集 | /expenses/:id/edit | 申請編集フォーム | 申請者 |
| SCR-006 | 経費精算申請詳細 | /expenses/:id | 申請詳細表示 | 権限者 |
| SCR-007 | 承認待ち一覧 | /approvals | 承認待ち申請一覧 | MANAGER |
| SCR-008 | 承認画面 | /approvals/:id | 承認・差戻し操作 | MANAGER |
| SCR-009 | ダッシュボード | /dashboard | 集計・統計表示 | MANAGER, ACCOUNTING, ADMIN |
| SCR-010 | ユーザー管理 | /admin/users | ユーザー一覧・編集 | ADMIN |
| SCR-011 | 承認ルート管理 | /admin/routes | 承認ルート設定 | ADMIN |
| SCR-012 | 代理設定管理 | /admin/delegations | 代理設定一覧・編集 | ADMIN |
| SCR-013 | 監査ログ | /admin/audit-logs | 監査ログ検索・閲覧 | ACCOUNTING, ADMIN |
| SCR-014 | 設定画面 | /settings | 表示言語設定 | 全員 |

## 2. 画面遷移図

```
[ログイン] --> [ホーム]
                 |
    +------------+------------+------------+
    |            |            |            |
    v            v            v            v
[申請一覧]  [承認待ち]  [ダッシュボード]  [管理]
    |            |                    |
    +---+        +---+           +----+----+
    |   |        |   |           |    |    |
    v   v        v   v           v    v    v
[作成][詳細]  [承認画面]     [ユーザー][ルート][代理]
    |                              |
    v                              v
[編集]                        [監査ログ]
```

## 3. 主要画面のコンポーネント構成

### 3.1 SCR-004: 経費精算申請作成画面

```
ExpenseCreatePage
+-- PageHeader（タイトル: "経費精算申請"）
+-- ExpenseForm
|   +-- BasicInfoSection
|   |   +-- TextField（タイトル）
|   +-- ExpenseItemsSection
|   |   +-- ExpenseItemRow（繰り返し）
|   |   |   +-- Select（経費種別）
|   |   |   +-- DatePicker（発生日）
|   |   |   +-- TextField（摘要）
|   |   |   +-- NumberField（金額）
|   |   |   +-- IconButton（削除）
|   |   +-- Button（明細行追加）
|   +-- TotalAmountDisplay（合計金額表示）
|   +-- AttachmentSection
|   |   +-- FileUploader（ドラッグ&ドロップ対応）
|   |   +-- AttachmentList
|   |       +-- AttachmentItem（繰り返し）
|   +-- ActionButtons
|       +-- Button（下書き保存）
|       +-- Button（申請提出）
|       +-- Button（キャンセル）
```

### 3.2 SCR-008: 承認画面

```
ApprovalPage
+-- PageHeader（タイトル: "承認"）
+-- ExpenseDetailView
|   +-- BasicInfoView（申請者・申請日・ステータス）
|   +-- ExpenseItemsTable（明細一覧テーブル）
|   +-- TotalAmountDisplay
|   +-- AttachmentListView（添付ファイル一覧）
+-- ApprovalHistoryView（承認履歴）
+-- ApprovalActionSection
    +-- TextField（コメント）
    +-- Button（承認）
    +-- Button（差戻し）
```

### 3.3 SCR-009: ダッシュボード画面

```
DashboardPage
+-- PageHeader（タイトル: "ダッシュボード"）
+-- FilterSection
|   +-- DateRangePicker（期間）
|   +-- Select（部門）
|   +-- Select（集計単位: 日次/週次/月次）
+-- SummaryCards
|   +-- StatCard（申請件数）
|   +-- StatCard（承認済み件数）
|   +-- StatCard（滞留件数）
|   +-- StatCard（合計金額）
+-- ChartsSection
|   +-- BarChart（期間別申請件数）
|   +-- PieChart（経費種別別金額）
+-- StaleApplicationsTable（滞留申請一覧）
```

## 4. フォームバリデーションルール

### 4.1 経費精算申請フォーム

| フィールド | バリデーション | タイミング |
|-----------|-------------|----------|
| タイトル | 必須、200文字以内 | onBlur |
| 経費種別 | 必須選択 | onChange |
| 発生日 | 必須、過去日 | onChange |
| 摘要 | 必須、500文字以内 | onBlur |
| 金額 | 必須、1円以上の整数 | onBlur |
| 合計金額 | 100万円以下 | 明細変更時に自動計算・チェック |
| 添付ファイル | 5MB/ファイル、合計10MB、PDF/JPEG/PNG | アップロード時 |

### 4.2 バリデーション表示
- フィールドレベル: 入力欄の下にエラーメッセージ表示（赤色）
- フォームレベル: 提出ボタン押下時に全フィールドバリデーション実行
- サーバーエラー: トースト通知で表示

## 5. API連携ポイント

### 5.1 画面別API呼出し

| 画面 | API | タイミング |
|------|-----|----------|
| ホーム | GET /api/v1/expenses（自分の最新申請） | 画面表示時 |
| ホーム | GET /api/v1/approvals/pending（承認待ち件数） | 画面表示時 |
| 申請一覧 | GET /api/v1/expenses | 画面表示時・検索時 |
| 申請作成 | POST /api/v1/expenses | 下書き保存・提出時 |
| 申請作成 | POST /api/v1/files/upload | ファイルアップロード時 |
| 申請編集 | GET /api/v1/expenses/{id} | 画面表示時 |
| 申請編集 | PUT /api/v1/expenses/{id} | 保存・提出時 |
| 申請詳細 | GET /api/v1/expenses/{id} | 画面表示時 |
| 申請詳細 | GET /api/v1/approvals/{id}/history | 画面表示時 |
| 承認待ち | GET /api/v1/approvals/pending | 画面表示時 |
| 承認画面 | GET /api/v1/expenses/{id} | 画面表示時 |
| 承認画面 | POST /api/v1/approvals/{id}/approve | 承認時 |
| 承認画面 | POST /api/v1/approvals/{id}/reject | 差戻し時 |
| ダッシュボード | GET /api/v1/dashboard/summary | 画面表示時 |
| ダッシュボード | GET /api/v1/dashboard/departments | 画面表示時 |
| ダッシュボード | GET /api/v1/dashboard/stale | 画面表示時 |
| ユーザー管理 | GET /api/v1/users | 画面表示時 |
| ユーザー管理 | POST/PUT /api/v1/users | 登録・更新時 |
| 承認ルート管理 | GET /api/v1/approval-routes | 画面表示時 |
| 監査ログ | GET /api/v1/audit-logs | 画面表示時・検索時 |

### 5.2 状態管理（Zustand）ストア構成

| ストア | 管理する状態 |
|--------|------------|
| authStore | 認証状態、ユーザー情報、トークン |
| expenseStore | 申請一覧、申請詳細、フォーム状態 |
| approvalStore | 承認待ち一覧、承認操作状態 |
| dashboardStore | 集計データ、フィルター条件 |
| uiStore | 言語設定、ローディング状態、通知 |
