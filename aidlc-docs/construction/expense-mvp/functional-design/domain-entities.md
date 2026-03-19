# ドメインエンティティ定義 — expense-mvp

## 1. エンティティ一覧

### 1.1 User（ユーザー）
| 属性名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| id | UUID | ○ | ユーザーID（主キー） |
| employee_number | VARCHAR(20) | ○ | 社員番号（一意） |
| email | VARCHAR(255) | ○ | メールアドレス（一意） |
| name_ja | VARCHAR(100) | ○ | 氏名（日本語） |
| name_en | VARCHAR(100) | - | 氏名（英語） |
| department_id | UUID | ○ | 所属部門ID（FK: Department） |
| supervisor_id | UUID | - | 直属上長ID（FK: User） |
| preferred_language | VARCHAR(5) | ○ | 表示言語（ja / en）、デフォルト: ja |
| is_active | BOOLEAN | ○ | 有効フラグ、デフォルト: true |
| created_at | TIMESTAMP | ○ | 作成日時 |
| updated_at | TIMESTAMP | ○ | 更新日時 |

### 1.2 Role（ロール）
| 属性名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| id | UUID | ○ | ロールID（主キー） |
| name | VARCHAR(50) | ○ | ロール名（一意） |
| description | VARCHAR(255) | - | ロール説明 |

### 1.3 UserRole（ユーザーロール）
| 属性名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| id | UUID | ○ | 主キー |
| user_id | UUID | ○ | ユーザーID（FK: User） |
| role_id | UUID | ○ | ロールID（FK: Role） |
| assigned_at | TIMESTAMP | ○ | 割当日時 |

### 1.4 Department（部門）
| 属性名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| id | UUID | ○ | 部門ID（主キー） |
| code | VARCHAR(20) | ○ | 部門コード（一意） |
| name_ja | VARCHAR(100) | ○ | 部門名（日本語） |
| name_en | VARCHAR(100) | - | 部門名（英語） |
| parent_id | UUID | - | 親部門ID（FK: Department） |
| manager_id | UUID | - | 部門長ID（FK: User） |
| is_active | BOOLEAN | ○ | 有効フラグ |
| created_at | TIMESTAMP | ○ | 作成日時 |
| updated_at | TIMESTAMP | ○ | 更新日時 |

### 1.5 Expense（経費精算申請）
| 属性名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| id | UUID | ○ | 申請ID（主キー） |
| application_number | VARCHAR(20) | ○ | 申請番号（一意、自動採番） |
| applicant_id | UUID | ○ | 申請者ID（FK: User） |
| proxy_applicant_id | UUID | - | 代理申請者ID（FK: User） |
| title | VARCHAR(200) | ○ | 申請タイトル |
| status | VARCHAR(20) | ○ | ステータス |
| total_amount | DECIMAL(12,0) | ○ | 合計金額 |
| department_id | UUID | ○ | 申請時の所属部門ID（FK: Department） |
| submitted_at | TIMESTAMP | - | 提出日時 |
| approved_at | TIMESTAMP | - | 最終承認日時 |
| journal_exported | BOOLEAN | ○ | 仕訳出力済みフラグ、デフォルト: false |
| created_at | TIMESTAMP | ○ | 作成日時 |
| updated_at | TIMESTAMP | ○ | 更新日時 |

### 1.6 ExpenseItem（経費明細行）
| 属性名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| id | UUID | ○ | 明細ID（主キー） |
| expense_id | UUID | ○ | 申請ID（FK: Expense） |
| expense_type_id | UUID | ○ | 経費種別ID（FK: ExpenseType） |
| description | VARCHAR(500) | ○ | 摘要 |
| amount | DECIMAL(12,0) | ○ | 金額 |
| incurred_date | DATE | ○ | 発生日 |
| sort_order | INTEGER | ○ | 表示順 |
| created_at | TIMESTAMP | ○ | 作成日時 |
| updated_at | TIMESTAMP | ○ | 更新日時 |

### 1.7 ExpenseType（経費種別）
| 属性名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| id | UUID | ○ | 経費種別ID（主キー） |
| code | VARCHAR(20) | ○ | 経費種別コード（一意） |
| name_ja | VARCHAR(50) | ○ | 経費種別名（日本語） |
| name_en | VARCHAR(50) | - | 経費種別名（英語） |
| account_code | VARCHAR(20) | ○ | 勘定科目コード |
| is_active | BOOLEAN | ○ | 有効フラグ |
| sort_order | INTEGER | ○ | 表示順 |

### 1.8 Attachment（添付ファイル）
| 属性名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| id | UUID | ○ | 添付ファイルID（主キー） |
| expense_id | UUID | ○ | 申請ID（FK: Expense） |
| file_name | VARCHAR(255) | ○ | ファイル名 |
| file_size | INTEGER | ○ | ファイルサイズ（バイト） |
| content_type | VARCHAR(100) | ○ | MIMEタイプ |
| s3_key | VARCHAR(500) | ○ | S3オブジェクトキー |
| uploaded_at | TIMESTAMP | ○ | アップロード日時 |
| uploaded_by | UUID | ○ | アップロード者ID（FK: User） |

### 1.9 ApprovalRoute（承認ルート）
| 属性名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| id | UUID | ○ | 承認ルートID（主キー） |
| department_id | UUID | ○ | 部門ID（FK: Department） |
| step_order | INTEGER | ○ | 承認段階（1: 直属上長、2: 部門長） |
| approver_id | UUID | ○ | 承認者ID（FK: User） |
| valid_from | DATE | ○ | 有効開始日 |
| valid_to | DATE | - | 有効終了日 |
| created_at | TIMESTAMP | ○ | 作成日時 |
| updated_at | TIMESTAMP | ○ | 更新日時 |

### 1.10 ApprovalRecord（承認レコード）
| 属性名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| id | UUID | ○ | 承認レコードID（主キー） |
| expense_id | UUID | ○ | 申請ID（FK: Expense） |
| step_order | INTEGER | ○ | 承認段階 |
| approver_id | UUID | ○ | 承認者ID（FK: User） |
| proxy_approver_id | UUID | - | 代理承認者ID（FK: User） |
| action | VARCHAR(20) | - | アクション（APPROVED / REJECTED） |
| comment | TEXT | - | コメント |
| acted_at | TIMESTAMP | - | 処理日時 |
| created_at | TIMESTAMP | ○ | 作成日時 |

### 1.11 DelegationSetting（代理設定）
| 属性名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| id | UUID | ○ | 代理設定ID（主キー） |
| delegator_id | UUID | ○ | 被代理者ID（FK: User） |
| delegate_id | UUID | ○ | 代理者ID（FK: User） |
| start_date | DATE | ○ | 開始日 |
| end_date | DATE | ○ | 終了日 |
| is_active | BOOLEAN | ○ | 有効フラグ |
| created_at | TIMESTAMP | ○ | 作成日時 |
| updated_at | TIMESTAMP | ○ | 更新日時 |

### 1.12 AuditLog（監査ログ）
| 属性名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| id | UUID | ○ | ログID（主キー） |
| user_id | UUID | ○ | 操作者ID（FK: User） |
| action | VARCHAR(50) | ○ | 操作種別 |
| resource_type | VARCHAR(50) | ○ | 対象リソース種別 |
| resource_id | UUID | ○ | 対象リソースID |
| details | JSONB | - | 変更前後の値（JSON） |
| ip_address | VARCHAR(45) | - | IPアドレス |
| created_at | TIMESTAMP | ○ | 操作日時 |

### 1.13 NotificationLog（通知ログ）
| 属性名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| id | UUID | ○ | 通知ログID（主キー） |
| recipient_id | UUID | ○ | 受信者ID（FK: User） |
| notification_type | VARCHAR(50) | ○ | 通知種別 |
| subject | VARCHAR(500) | ○ | 件名 |
| status | VARCHAR(20) | ○ | 送信ステータス（SENT / FAILED） |
| error_message | TEXT | - | エラーメッセージ |
| sent_at | TIMESTAMP | ○ | 送信日時 |

### 1.14 BatchExecutionLog（バッチ実行ログ）
| 属性名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| id | UUID | ○ | ログID（主キー） |
| batch_type | VARCHAR(50) | ○ | バッチ種別 |
| status | VARCHAR(20) | ○ | 実行ステータス（SUCCESS / FAILED / PARTIAL） |
| total_records | INTEGER | ○ | 総レコード数 |
| success_count | INTEGER | ○ | 成功件数 |
| error_count | INTEGER | ○ | エラー件数 |
| error_file_s3_key | VARCHAR(500) | - | エラーファイルのS3キー |
| started_at | TIMESTAMP | ○ | 開始日時 |
| completed_at | TIMESTAMP | - | 完了日時 |

## 2. ER図

```
+----------+     +----------+     +-----------+
|   User   |---->|Department|<----|ApprovalRte|
+----------+     +----------+     +-----------+
  |  |  |                              |
  |  |  +------+                       |
  |  |         |                       |
  |  v         v                       v
+------+  +----------+         +----------+
| Role |  |Delegation|         |ApprovalRec|
+------+  | Setting  |         +----------+
  |       +----------+              |
  v                                 |
+----------+                        |
| UserRole |                        |
+----------+                        |
                                    |
+----------+     +-----------+      |
| Expense  |---->|ExpenseItem|      |
+----------+     +-----------+      |
  |    |              |             |
  |    |              v             |
  |    |         +-----------+      |
  |    |         |ExpenseType|      |
  |    |         +-----------+      |
  |    |                            |
  |    +-------->+----------+       |
  |              |Attachment|       |
  |              +----------+       |
  |                                 |
  +---------------------------------+

+----------+  +----------+  +----------+
| AuditLog |  |NotifyLog |  |BatchLog  |
+----------+  +----------+  +----------+
```

### 主要リレーションシップ

| 親エンティティ | 子エンティティ | 関係 | 説明 |
|--------------|--------------|------|------|
| User | Expense | 1:N | ユーザーは複数の申請を持つ |
| Expense | ExpenseItem | 1:N | 申請は複数の明細行を持つ |
| Expense | Attachment | 1:N | 申請は複数の添付ファイルを持つ |
| Expense | ApprovalRecord | 1:N | 申請は複数の承認レコードを持つ |
| ExpenseType | ExpenseItem | 1:N | 経費種別は複数の明細行で使用 |
| Department | User | 1:N | 部門は複数のユーザーを持つ |
| Department | ApprovalRoute | 1:N | 部門は複数の承認ルートを持つ |
| Department | Department | 1:N | 部門は親子関係を持つ（自己参照） |
| User | DelegationSetting | 1:N | ユーザーは複数の代理設定を持つ |
| User | Role (via UserRole) | N:M | ユーザーとロールは多対多 |
