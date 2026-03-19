# コンポーネント依存関係

## 依存関係マトリクス

凡例: ◎ = 直接依存、○ = 間接依存、- = 依存なし

### バックエンドサービス間依存関係

| 呼出し元 ＼ 呼出し先 | Expense | Workflow | User | Route | Dashboard | File | Notification | Audit | Batch | Auth |
|---------------------|---------|----------|------|-------|-----------|------|-------------|-------|-------|------|
| ExpenseService      | -       | ◎       | -    | -     | -         | ◎   | -           | ◎    | -     | -    |
| WorkflowService     | -       | -        | ◎   | ◎    | -         | -    | ◎          | ◎    | -     | -    |
| UserService         | -       | -        | -    | -     | -         | -    | -           | ◎    | -     | -    |
| RouteService        | -       | -        | ◎   | -     | -         | -    | -           | ◎    | -     | -    |
| DashboardService    | -       | -        | -    | -     | -         | -    | -           | -     | -     | -    |
| FileService         | -       | -        | -    | -     | -         | -    | -           | ◎    | -     | -    |
| NotificationService | -       | -        | -    | -     | -         | -    | -           | -     | -     | -    |
| AuditService        | -       | -        | -    | -     | -         | -    | -           | -     | -     | -    |
| BatchService        | -       | -        | -    | -     | -         | -    | -           | ◎    | -     | -    |
| AuthService         | -       | -        | ◎   | -     | -         | -    | -           | -     | -     | -    |

### フロントエンド → バックエンドAPI依存関係

| フロントエンド ＼ バックエンドAPI | Expense | Workflow | User | Route | Dashboard | File | Audit | Auth |
|-------------------------------|---------|----------|------|-------|-----------|------|-------|------|
| FE-AUTH                       | -       | -        | ◎   | -     | -         | -    | -     | ◎   |
| FE-EXPENSE                    | ◎      | -        | -    | -     | -         | ◎   | -     | -    |
| FE-APPROVAL                   | ◎      | ◎       | -    | -     | -         | ◎   | -     | -    |
| FE-INQUIRY                    | ◎      | ◎       | -    | -     | -         | -    | -     | -    |
| FE-DASHBOARD                  | -       | -        | -    | -     | ◎        | -    | -     | -    |
| FE-ADMIN                      | -       | -        | ◎   | ◎    | -         | -    | ◎    | -    |

## 通信パターン

### フロントエンド ↔ バックエンド
- **プロトコル**: HTTPS（REST API）
- **認証**: Bearer Token（OIDC アクセストークン）
- **データ形式**: JSON
- **API Gateway**: Amazon API Gateway（Lambda プロキシ統合）

### バックエンド ↔ データベース
- **プロトコル**: PostgreSQL プロトコル（TLS暗号化）
- **ORM**: SQLAlchemy
- **接続管理**: コネクションプーリング

### バックエンド ↔ 外部サービス
- **Amazon S3**: AWS SDK（boto3）経由、HTTPS
- **Amazon SES**: AWS SDK（boto3）経由、HTTPS
- **外部IdP**: OIDC/SAML プロトコル、HTTPS

### バッチ処理
- **トリガー**: スケジュール実行（EventBridge / CloudWatch Events）
- **入力**: CSVファイル（S3バケットから取得）
- **出力**: CSVファイル（S3バケットへ出力）

## データフロー図

### 経費精算申請フロー
```
社員 --> FE-EXPENSE --> API Gateway --> BE-EXPENSE
                                            |
                                    +-------+-------+
                                    |               |
                              BE-FILE          BE-WORKFLOW
                                |                   |
                              S3               BE-USER
                                                    |
                                              BE-ROUTE
                                                    |
                                            BE-NOTIFICATION
                                                    |
                                              Amazon SES
                                                    |
                                              承認者メール
```

### 承認フロー
```
承認者 --> FE-APPROVAL --> API Gateway --> BE-WORKFLOW
                                              |
                                      +-------+-------+
                                      |               |
                                BE-EXPENSE      BE-NOTIFICATION
                                                      |
                                                Amazon SES
                                                      |
                                                申請者メール
```

### バッチ連携フロー
```
人事システム --> CSVファイル --> S3 --> BE-BATCH --> PostgreSQL
                                                      |
組織システム --> CSVファイル --> S3 --> BE-BATCH --------+

PostgreSQL --> BE-BATCH --> CSVファイル --> S3 --> 会計システム
```

### 認証フロー
```
社員 --> FE-AUTH --> 外部IdP（Azure AD / Okta）
                        |
                   OIDCコールバック
                        |
              API Gateway --> BE-AUTH --> PostgreSQL
                                |
                          トークン発行
                                |
                          FE-AUTH（トークン保存）
```
