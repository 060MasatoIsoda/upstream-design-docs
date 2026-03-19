# NFR設計パターン — expense-mvp

## 1. 認証・認可パターン

### 1.1 OIDC認証フロー（Authorization Code Flow + PKCE）

```
ブラウザ          フロントエンド        API Gateway       Lambda         IdP
  |                   |                   |               |              |
  |--[ログイン]------>|                   |               |              |
  |                   |--[認可リクエスト（PKCE）]------------------------>|
  |                   |                   |               |              |
  |<--[リダイレクト]--|                   |               |              |
  |--[認証情報入力]-------------------------------------------------->|
  |<--[認可コード]---------------------------------------------------|
  |                   |                   |               |              |
  |--[認可コード]--->|                   |               |              |
  |                   |--[トークン交換]------------------>[callback]    |
  |                   |                   |               |--[検証]---->|
  |                   |                   |               |<-[トークン]-|
  |                   |<--[JWT]----------|               |              |
  |<--[認証完了]------|                   |               |              |
```

### 1.2 APIリクエスト認可パターン

```python
# パターン: FastAPI Dependency Injection による認可

# 1. トークン検証ミドルウェア（全APIに適用）
async def verify_token(authorization: str = Header()) -> TokenPayload:
    """JWTトークンを検証し、ペイロードを返す"""
    # JWKS検証、有効期限チェック、issuer/audience検証

# 2. ロールチェックデコレータ
def require_role(*roles: str):
    """指定ロールを持つユーザーのみアクセス許可"""
    # TokenPayloadからロールを取得し検証

# 3. オブジェクトレベル認可
async def verify_expense_access(expense_id: UUID, current_user: User):
    """申請データへのアクセス権限を検証"""
    # 申請者本人、承認者、代理承認者、経理、管理者のみ許可
```

### 1.3 RBAC実装パターン

```
リクエスト
    |
    v
[トークン検証] --> 無効 --> 401 Unauthorized
    |
    有効
    |
    v
[ロールチェック] --> 権限なし --> 403 Forbidden
    |
    権限あり
    |
    v
[オブジェクトレベル認可] --> アクセス不可 --> 403 Forbidden
    |
    アクセス可
    |
    v
[ビジネスロジック実行]
```

## 2. エラーハンドリング・レジリエンスパターン

### 2.1 グローバルエラーハンドラー

```python
# パターン: FastAPI Exception Handler

# 1. グローバルエラーハンドラー（全未処理例外をキャッチ）
@app.exception_handler(Exception)
async def global_exception_handler(request, exc):
    """未処理例外をキャッチし、安全なレスポンスを返す"""
    # ログ記録（structlog）
    # 汎用エラーメッセージ返却（内部情報を含めない）
    # 500 Internal Server Error

# 2. ビジネス例外ハンドラー
@app.exception_handler(BusinessException)
async def business_exception_handler(request, exc):
    """ビジネスルール違反を適切なHTTPステータスで返す"""
    # 400 Bad Request / 409 Conflict 等

# 3. バリデーション例外ハンドラー
@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request, exc):
    """Pydanticバリデーションエラーを整形して返す"""
    # 422 Unprocessable Entity
```

### 2.2 Fail-Closed設計

| シナリオ | 挙動 |
|---------|------|
| トークン検証失敗 | アクセス拒否（401） |
| ロールチェック失敗 | アクセス拒否（403） |
| DB接続エラー | 500エラー返却、リトライなし |
| S3アクセスエラー | 500エラー返却、ファイル操作中断 |
| SES送信エラー | エラーログ記録、通知失敗を記録（申請処理は継続） |
| 外部IdP応答なし | 認証失敗（タイムアウト） |

### 2.3 リトライパターン

| 対象 | リトライ | 回数 | 間隔 |
|------|---------|------|------|
| DB接続 | あり | 3回 | 指数バックオフ（1s, 2s, 4s） |
| S3操作 | あり | 3回 | 指数バックオフ（boto3デフォルト） |
| SES送信 | あり | 2回 | 固定間隔（5s） |
| IdP通信 | なし | - | タイムアウト10秒 |

## 3. ログ・モニタリングパターン

### 3.1 構造化ログパターン

```python
# パターン: structlog + aws-lambda-powertools

# ログ出力形式（JSON）
{
    "timestamp": "2026-03-19T10:30:00.000Z",
    "level": "INFO",
    "message": "経費精算申請を作成",
    "correlation_id": "req-abc123",
    "user_id": "usr-def456",
    "resource_type": "expense",
    "resource_id": "exp-ghi789",
    "action": "create",
    "duration_ms": 150,
    "lambda_request_id": "lambda-xxx"
}
```

### 3.2 相関IDパターン

```
フロントエンド                API Gateway              Lambda
    |                           |                       |
    |--[X-Correlation-ID]------>|                       |
    |                           |--[correlation_id]---->|
    |                           |                       |--[ログ出力]
    |                           |                       |  correlation_id付与
    |                           |                       |
    |                           |<--[レスポンス]---------|
    |<--[X-Correlation-ID]------|                       |
```

- フロントエンドでUUID v4を生成し、リクエストヘッダーに付与
- Lambda内で全ログに相関IDを付与
- エラー発生時、相関IDでログを横断検索可能

### 3.3 監査ログパターン

```python
# パターン: AuditService による一元管理

# 監査ログ記録（全サービスから呼出し）
async def record_audit(
    user_id: UUID,
    action: str,           # "CREATE", "UPDATE", "APPROVE", "REJECT" 等
    resource_type: str,    # "expense", "user", "approval_route" 等
    resource_id: UUID,
    details: dict = None,  # 変更前後の値
    ip_address: str = None
):
    """操作ログをDBに記録"""
```

## 4. 性能最適化パターン

### 4.1 データベースクエリ最適化

| パターン | 適用箇所 | 説明 |
|---------|---------|------|
| インデックス設計 | 検索・フィルタ対象カラム | 複合インデックス含む |
| ページネーション | 一覧API全般 | OFFSET/LIMIT方式 |
| Eager Loading | 申請詳細取得 | N+1問題回避（joinedload） |
| 読み取り専用クエリ | ダッシュボード集計 | 読み取り専用トランザクション |

### 4.2 主要インデックス設計

| テーブル | インデックス | カラム |
|---------|------------|--------|
| expenses | idx_expenses_applicant_status | (applicant_id, status) |
| expenses | idx_expenses_department_status | (department_id, status) |
| expenses | idx_expenses_submitted_at | (submitted_at) |
| expenses | idx_expenses_status_journal | (status, journal_exported) |
| approval_records | idx_approvals_approver_action | (approver_id, action) |
| approval_records | idx_approvals_expense | (expense_id, step_order) |
| audit_logs | idx_audit_created_at | (created_at) |
| audit_logs | idx_audit_resource | (resource_type, resource_id) |
| users | idx_users_department | (department_id) |
| users | idx_users_employee_number | (employee_number) UNIQUE |

### 4.3 Lambda コールドスタート対策

| 対策 | 説明 |
|------|------|
| Provisioned Concurrency | 業務時間帯（9:00-18:00）に最小5インスタンス確保（将来検討） |
| 依存関係最小化 | 不要なライブラリを除外 |
| Lambdaレイヤー | 共通ライブラリをレイヤーに分離 |
| 接続プール | DB接続をLambda実行環境で再利用 |

### 4.4 フロントエンド最適化

| パターン | 説明 |
|---------|------|
| コード分割 | React.lazy + Suspense による画面単位の遅延読込 |
| キャッシュ | TanStack Query によるサーバー状態キャッシュ（staleTime設定） |
| バンドル最適化 | Vite Tree Shaking、MUIのバレルインポート回避 |
| 画像最適化 | 添付ファイルプレビューのサムネイル生成（将来検討） |

## 5. セキュリティパターン

### 5.1 入力バリデーション多層防御

```
フロントエンド（Zod）
    |
    v  [クライアントサイドバリデーション]
    |
API Gateway
    |
    v  [リクエストサイズ制限、スロットリング]
    |
Lambda（Pydantic）
    |
    v  [サーバーサイドバリデーション]
    |
Repository（SQLAlchemy）
    |
    v  [パラメータ化クエリ、型制約]
    |
PostgreSQL
    v  [DB制約、CHECK制約]
```

### 5.2 レート制限

| 対象 | 制限 | 方式 |
|------|------|------|
| API全体 | 1,000リクエスト/秒 | API Gatewayスロットリング |
| 認証エンドポイント | 10リクエスト/分/IP | API Gateway使用量プラン |
| ファイルアップロード | 10リクエスト/分/ユーザー | アプリケーション層 |

### 5.3 WAFルール

| ルール | 説明 |
|--------|------|
| AWS Managed Rules - Common | 一般的な攻撃パターン防御 |
| AWS Managed Rules - SQL Injection | SQLインジェクション防御 |
| AWS Managed Rules - Known Bad Inputs | 既知の悪意あるリクエスト防御 |
| レート制限ルール | IP単位のリクエスト制限 |
| 地理的制限 | 日本国内からのアクセスのみ許可（オプション） |

## 6. データ保護パターン

### 6.1 暗号化レイヤー

```
+------------------------------------------+
|          通信暗号化（TLS 1.2+）            |
|  CloudFront <-> ブラウザ                   |
|  API Gateway <-> Lambda                   |
|  Lambda <-> RDS（SSL/TLS）                |
|  Lambda <-> S3（HTTPS）                   |
+------------------------------------------+
|          保存時暗号化                       |
|  RDS: AES-256（AWS KMS管理キー）           |
|  S3: SSE-S3（AES-256）                    |
|  Secrets Manager: AWS KMS                 |
+------------------------------------------+
```

### 6.2 機密情報管理

| 機密情報 | 保管先 | アクセス方式 |
|---------|--------|------------|
| DB接続文字列 | Secrets Manager | Lambda環境変数（暗号化） |
| IdPクライアントシークレット | Secrets Manager | Lambda実行時に取得 |
| JWKSエンドポイント | 環境変数 | Lambda環境変数 |
| S3バケット名 | 環境変数 | Lambda環境変数 |
| SES送信元アドレス | 環境変数 | Lambda環境変数 |
