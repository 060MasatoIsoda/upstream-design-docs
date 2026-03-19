---
layout: default
title: サービス定義
---

# サービス定義

## アーキテクチャ概要

レイヤードアーキテクチャ（3層構造）を採用する。

```
+---------------------------------------------------+
|              フロントエンド（React）                  |
|  FE-AUTH | FE-EXPENSE | FE-APPROVAL | FE-INQUIRY  |
|  FE-DASHBOARD | FE-ADMIN | FE-COMMON              |
+---------------------------------------------------+
                        |
                   REST API (HTTPS)
                        |
+---------------------------------------------------+
|           API Gateway (Amazon API Gateway)          |
+---------------------------------------------------+
                        |
                   Lambda関数
                        |
+---------------------------------------------------+
|              Router層（FastAPI Router）              |
|  リクエスト受付、バリデーション、レスポンス整形        |
+---------------------------------------------------+
                        |
+---------------------------------------------------+
|              Service層（ビジネスロジック）             |
|  ExpenseService | WorkflowService | UserService    |
|  RouteService | DashboardService | FileService     |
|  NotificationService | AuditService | BatchService |
+---------------------------------------------------+
                        |
+---------------------------------------------------+
|              Repository層（データアクセス）           |
|  SQLAlchemy ORM + Alembic マイグレーション           |
+---------------------------------------------------+
                        |
+---------------------------------------------------+
|              PostgreSQL データベース                  |
+---------------------------------------------------+
```

## サービス一覧

### 1. ExpenseService（経費精算サービス）
- **責務**: 経費精算申請のライフサイクル管理
- **依存先**: Repository層、WorkflowService、AuditService、FileService
- **オーケストレーション**:
  - 申請提出時: バリデーション → 申請保存 → WorkflowServiceで承認ルート生成 → NotificationServiceで承認依頼通知 → AuditServiceでログ記録

### 2. WorkflowService（承認ワークフローサービス）
- **責務**: 承認プロセスの管理とステータス遷移
- **依存先**: Repository層、UserService、NotificationService、AuditService
- **オーケストレーション**:
  - 承認時: 権限チェック → 承認記録 → 次段階判定 → ステータス更新 → 通知送信 → ログ記録
  - 差戻し時: 権限チェック → 差戻し記録 → ステータス更新 → 通知送信 → ログ記録

### 3. UserService（ユーザー管理サービス）
- **責務**: ユーザー情報・ロール・組織情報の管理
- **依存先**: Repository層、AuditService
- **オーケストレーション**:
  - ユーザー操作時: 権限チェック → データ操作 → ログ記録

### 4. RouteService（承認ルート管理サービス）
- **責務**: 承認ルートの設定・判定
- **依存先**: Repository層、UserService、AuditService
- **オーケストレーション**:
  - ルート判定時: 申請者の所属部門取得 → 直属上長取得 → 部門長取得 → ルート生成

### 5. DashboardService（ダッシュボードサービス）
- **責務**: 集計データの生成
- **依存先**: Repository層
- **オーケストレーション**:
  - 集計時: 期間・部門条件の解析 → 集計クエリ実行 → 結果整形

### 6. FileService（ファイル管理サービス）
- **責務**: 添付ファイルの管理
- **依存先**: Amazon S3、Repository層、AuditService
- **オーケストレーション**:
  - アップロード時: サイズ・形式バリデーション → S3アップロード → メタデータ保存 → ログ記録
  - ダウンロード時: 権限チェック → 署名付きURL生成

### 7. NotificationService（通知サービス）
- **責務**: メール通知の送信
- **依存先**: Amazon SES、Repository層
- **オーケストレーション**:
  - 通知送信時: テンプレート取得 → 宛先解決 → メール送信 → 送信履歴記録

### 8. AuditService（監査ログサービス）
- **責務**: 操作ログの記録と検索
- **依存先**: Repository層
- **オーケストレーション**:
  - ログ記録時: 操作情報の構造化 → データベース保存

### 9. AuthService（認証サービス）
- **責務**: OIDC認証フローの処理、トークン管理
- **依存先**: 外部IdP、Repository層
- **オーケストレーション**:
  - 認証時: IdPからのコールバック受信 → トークン検証 → ユーザー情報取得/作成 → セッション生成

### 10. BatchService（バッチ処理サービス）
- **責務**: 外部システムとのデータ連携
- **依存先**: Repository層、AuditService
- **オーケストレーション**:
  - マスタ取込時: ファイル読込 → バリデーション → 差分検出 → データ更新 → 結果ログ記録
  - 仕訳出力時: 対象データ抽出 → 仕訳変換 → ファイル出力 → 結果ログ記録

## サービス間相互作用

```
+------------------+       +-------------------+
|  ExpenseService  |------>|  WorkflowService  |
+------------------+       +-------------------+
        |                          |
        v                          v
+------------------+       +-------------------+
|   FileService    |       |   UserService     |
+------------------+       +-------------------+
        |                          |
        v                          v
+------------------+       +-------------------+
|   Amazon S3      |       |  RouteService     |
+------------------+       +-------------------+
                                   
+------------------+       +-------------------+
| DashboardService |       |NotificationService|
+------------------+       +-------------------+
                                   |
                                   v
                           +-------------------+
                           |   Amazon SES      |
                           +-------------------+

+------------------+
|  AuditService    |<------ 全サービスから呼出し
+------------------+

+------------------+
|  BatchService    |<------ スケジュール実行
+------------------+
```

### 呼出し関係サマリー

| 呼出し元 | 呼出し先 | 目的 |
|---------|---------|------|
| ExpenseService | WorkflowService | 申請提出時の承認ルート生成 |
| ExpenseService | FileService | 添付ファイルの関連付け |
| ExpenseService | AuditService | 操作ログ記録 |
| WorkflowService | UserService | 承認者情報の取得 |
| WorkflowService | NotificationService | 承認依頼・完了・差戻し通知 |
| WorkflowService | AuditService | 操作ログ記録 |
| RouteService | UserService | 上長・部門長情報の取得 |
| FileService | AuditService | 操作ログ記録 |
| UserService | AuditService | 操作ログ記録 |
| RouteService | AuditService | 操作ログ記録 |
| NotificationService | （Amazon SES） | メール送信 |
| FileService | （Amazon S3） | ファイル保存・取得 |
| BatchService | AuditService | バッチ実行ログ記録 |
