# デプロイアーキテクチャ — expense-mvp

## 1. 環境構成

### 1.1 環境一覧

| 環境 | 用途 | ドメイン（例） | デプロイ方式 |
|------|------|-------------|------------|
| dev | 開発・動作確認 | dev-expense.example.com | Push to dev ブランチで自動デプロイ |
| staging | 結合テスト・受入テスト | stg-expense.example.com | PR マージで自動デプロイ |
| production | 本番運用 | expense.example.com | 手動承認後デプロイ |

### 1.2 環境別リソースサイジング

| リソース | dev | staging | production |
|---------|-----|---------|------------|
| Lambda (api-handler) メモリ | 128MB | 256MB | 512MB |
| Lambda (api-handler) 同時実行 | 5 | 10 | 50 |
| Lambda (batch) メモリ | 256MB | 512MB | 512MB |
| RDS インスタンスクラス | db.t3.micro | db.t3.small | db.t3.medium |
| RDS マルチAZ | 無効 | 無効 | 有効 |
| RDS ストレージ | 20GB (固定) | 20GB (固定) | 20GB (自動拡張100GB) |
| RDS バックアップ保持 | 1日 | 3日 | 7日 |
| NAT Gateway | 1AZ (コスト削減) | 1AZ (コスト削減) | 2AZ (高可用性) |
| CloudFront WAF | 有効 | 有効 | 有効 |
| CloudWatch アラーム | 最小限 | 全項目 | 全項目 |
| RDS パフォーマンスインサイト | 無効 | 有効 | 有効 |
| RDS 削除保護 | 無効 | 無効 | 有効 |

### 1.3 環境別コスト概算（月額）

| 環境 | 概算コスト | 備考 |
|------|----------|------|
| dev | $80〜120 | NAT Gateway 1AZ、最小インスタンス |
| staging | $120〜180 | NAT Gateway 1AZ、中間インスタンス |
| production | $230〜340 | NAT Gateway 2AZ、マルチAZ RDS |

## 2. デプロイフロー

### 2.1 CI/CD パイプライン全体像

```
+-------+     +----------+     +---------+     +----------+
| Push  | --> | CI       | --> | Build   | --> | Deploy   |
| / PR  |     | チェック  |     | パッケージ|     | 環境別   |
+-------+     +----------+     +---------+     +----------+
                  |                 |                |
                  v                 v                v
              +--------+     +---------+     +----------+
              | Lint   |     | Lambda  |     | dev      |
              | Type   |     | ZIP     |     | staging  |
              | Test   |     | FE Build|     | prod     |
              +--------+     +---------+     +----------+
```

### 2.2 GitHub Actions ワークフロー

#### バックエンドパイプライン

| ステップ | トリガー | 処理内容 |
|---------|---------|---------|
| Lint & Type Check | Push / PR | Ruff lint、mypy型チェック |
| Unit Test | Push / PR | pytest実行、カバレッジ計測 |
| Build | PR マージ | Lambda ZIP パッケージ作成 |
| Deploy to dev | dev ブランチ Push | Terraform apply (dev) |
| Deploy to staging | main ブランチ PR マージ | Terraform apply (staging) |
| Deploy to production | リリースタグ + 手動承認 | Terraform apply (production) |

#### フロントエンドパイプライン

| ステップ | トリガー | 処理内容 |
|---------|---------|---------|
| Lint & Type Check | Push / PR | ESLint、TypeScript型チェック |
| Unit Test | Push / PR | Vitest実行、カバレッジ計測 |
| Build | PR マージ | Vite ビルド（環境別設定） |
| Deploy to dev | dev ブランチ Push | S3 sync + CloudFront invalidation |
| Deploy to staging | main ブランチ PR マージ | S3 sync + CloudFront invalidation |
| Deploy to production | リリースタグ + 手動承認 | S3 sync + CloudFront invalidation |

#### インフラパイプライン

| ステップ | トリガー | 処理内容 |
|---------|---------|---------|
| Terraform fmt | Push / PR | フォーマットチェック |
| Terraform validate | Push / PR | 構文検証 |
| Terraform plan | PR | 変更差分表示（PRコメント） |
| Terraform apply (dev) | dev ブランチ Push | 開発環境適用 |
| Terraform apply (staging) | main ブランチ PR マージ | ステージング環境適用 |
| Terraform apply (production) | リリースタグ + 手動承認 | 本番環境適用 |

### 2.3 デプロイ戦略

| 項目 | 設定 |
|------|------|
| Lambda デプロイ | 全トラフィック切替（エイリアス使用） |
| フロントエンド デプロイ | S3 sync + CloudFront キャッシュ無効化 |
| DB マイグレーション | Alembic upgrade（デプロイ前に実行） |
| ロールバック（Lambda） | 前バージョンのエイリアス切替 |
| ロールバック（フロントエンド） | 前バージョンのS3 sync |
| ロールバック（DB） | Alembic downgrade（破壊的変更は手動対応） |

## 3. ブランチ戦略

```
main (本番リリース用)
  |
  +-- develop (開発統合ブランチ)
  |     |
  |     +-- feature/xxx (機能開発)
  |     +-- bugfix/xxx (バグ修正)
  |
  +-- release/x.x.x (リリース準備)
  +-- hotfix/xxx (緊急修正)
```

| ブランチ | デプロイ先 | マージ先 |
|---------|----------|---------|
| feature/* | — (CIのみ) | develop |
| develop | dev 環境 | main (PR経由) |
| release/* | staging 環境 | main (PR経由) |
| main | production 環境 | — |
| hotfix/* | staging → production | main + develop |

## 4. Terraform バックエンド構成

### 4.1 状態管理

| 項目 | 設定 |
|------|------|
| バックエンド | S3 |
| 状態ファイルバケット | expense-terraform-state-{account-id} |
| ロックテーブル | DynamoDB (expense-terraform-lock) |
| 暗号化 | SSE-S3 |
| バージョニング | 有効 |

### 4.2 環境別ワークスペース

| 環境 | 状態ファイルキー | 変数ファイル |
|------|---------------|------------|
| dev | env/dev/terraform.tfstate | environments/dev/terraform.tfvars |
| staging | env/staging/terraform.tfstate | environments/staging/terraform.tfvars |
| production | env/production/terraform.tfstate | environments/production/terraform.tfvars |


## 5. セキュリティ考慮事項（デプロイ関連）

### 5.1 シークレット管理

| 項目 | 方式 |
|------|------|
| AWS認証情報 | GitHub Actions OIDC（IAMロール引き受け） |
| DB接続情報 | Secrets Manager（Terraform管理） |
| IdP設定 | Secrets Manager（手動設定） |
| 環境変数 | Terraform変数（機密値はSecrets Manager参照） |

### 5.2 デプロイ権限

| 環境 | 必要な権限 | 承認フロー |
|------|----------|----------|
| dev | 開発者IAMロール | 不要（自動デプロイ） |
| staging | CI/CD IAMロール | PR承認（1名以上） |
| production | CI/CD IAMロール（制限付き） | PR承認（2名以上）+ GitHub Environment承認 |

### 5.3 GitHub Actions セキュリティ

| 項目 | 設定 |
|------|------|
| AWS認証 | OIDC（長期キー不使用） |
| シークレット | GitHub Secrets（環境別） |
| 依存脆弱性 | Dependabot有効 |
| ワークフロー権限 | 最小権限の原則 |
| ブランチ保護 | main: PR必須、レビュー必須、ステータスチェック必須 |

## 6. 障害復旧

### 6.1 復旧目標

| 項目 | 目標値 |
|------|--------|
| RTO（目標復旧時間） | 4時間 |
| RPO（目標復旧時点） | 1時間 |

### 6.2 復旧手順概要

| 障害シナリオ | 復旧手順 | 想定復旧時間 |
|------------|---------|------------|
| Lambda障害 | 前バージョンへのエイリアス切替 | 5分 |
| フロントエンド障害 | 前バージョンのS3 sync | 10分 |
| RDS障害（単一AZ） | マルチAZフェイルオーバー（自動） | 1〜3分 |
| RDS障害（データ破損） | ポイントインタイムリカバリ | 30分〜1時間 |
| リージョン障害 | 手動復旧（バックアップからリストア） | 2〜4時間 |
| デプロイ失敗 | ロールバック手順実行 | 10〜30分 |

### 6.3 バックアップ戦略

| 対象 | 方式 | 保持期間 | 頻度 |
|------|------|---------|------|
| RDS | 自動バックアップ | 7日（production） | 日次 |
| RDS | スナップショット | 30日 | リリース前 |
| S3 (attachments) | バージョニング | 無期限 | 自動 |
| S3 (frontend) | バージョニング | 無期限 | 自動 |
| Terraform状態 | S3バージョニング | 無期限 | 変更時 |
