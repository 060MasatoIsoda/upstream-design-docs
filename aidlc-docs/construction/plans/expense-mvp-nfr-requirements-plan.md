---
layout: default
title: NFR要件計画
---

# NFR要件計画 — expense-mvp

## 評価スコープ
経費精算MVP（expense-mvp）ユニットの非機能要件を詳細化し、技術スタックの具体的な選定を行う。

## NFR要件計画チェックリスト

### 1. 性能要件の詳細化
- [x] API レスポンスタイム目標の定義
- [x] データベースクエリ性能の目標定義
- [x] バッチ処理性能の目標定義
- [x] フロントエンド表示性能の目標定義

### 2. セキュリティ要件の詳細化
- [x] SECURITY-01〜15 の適用方針定義
- [x] 認証・認可の実装方式詳細
- [x] データ暗号化方式の定義
- [x] セキュリティヘッダーの定義

### 3. 可用性・信頼性要件の詳細化
- [x] バックアップ・リストア方式の定義
- [x] 障害検知・復旧方式の定義
- [x] ログ・モニタリング方式の定義

### 4. 技術スタック詳細決定
- [x] Python / FastAPI の具体的ライブラリ選定
- [x] React / TypeScript の具体的ライブラリ選定
- [x] テストフレームワークの選定
- [x] CI/CD パイプラインツールの選定

---

## NFR要件に関する質問

以下の質問に回答してください。各質問の `[Answer]:` タグの後に選択肢の記号を記入してください。

### 質問 1: Pythonバックエンドのフレームワーク詳細
FastAPIと組み合わせるORMとして、SQLAlchemy のバージョンはどちらを使用しますか？

A) SQLAlchemy 2.x（最新、async対応強化、型ヒント改善）
B) SQLAlchemy 1.4（安定版、async対応あり）
C) 特に指定なし（AI-DLCに判断を委ねる）
X) Other（`[Answer]:` タグの後に詳細を記述してください）

[Answer]: A

### 質問 2: Lambda実行環境
AWS Lambda の実行環境として、どちらを使用しますか？

A) Lambda（Python ランタイム直接）— コールドスタートあり、シンプル
B) Lambda + コンテナイメージ — カスタムランタイム、依存関係の柔軟性
C) Lambda + Mangum（ASGIアダプター）— FastAPIをそのままLambdaで実行
D) 特に指定なし（AI-DLCに判断を委ねる）
X) Other（`[Answer]:` タグの後に詳細を記述してください）

[Answer]: A

### 質問 3: テストフレームワーク
テストフレームワークとして、どれを使用しますか？

A) バックエンド: pytest / フロントエンド: Vitest + React Testing Library
B) バックエンド: pytest / フロントエンド: Jest + React Testing Library
C) 特に指定なし（AI-DLCに判断を委ねる）
X) Other（`[Answer]:` タグの後に詳細を記述してください）

[Answer]: A

### 質問 4: CI/CDパイプライン
CI/CDパイプラインとして、どれを使用しますか？

A) AWS CodePipeline + CodeBuild
B) GitHub Actions
C) GitLab CI/CD
D) 特に指定なし（AI-DLCに判断を委ねる）
X) Other（`[Answer]:` タグの後に詳細を記述してください）

[Answer]: B

### 質問 5: IaC（Infrastructure as Code）ツール
インフラのコード管理ツールとして、どれを使用しますか？

A) AWS CDK（TypeScript）
B) AWS CDK（Python）
C) Terraform
D) AWS SAM（Serverless Application Model）
E) 特に指定なし（AI-DLCに判断を委ねる）
X) Other（`[Answer]:` タグの後に詳細を記述してください）

[Answer]: C

### 質問 6: PostgreSQLのホスティング
PostgreSQLのホスティング方式として、どれを使用しますか？

A) Amazon RDS for PostgreSQL
B) Amazon Aurora Serverless v2（PostgreSQL互換）
C) 特に指定なし（AI-DLCに判断を委ねる）
X) Other（`[Answer]:` タグの後に詳細を記述してください）

[Answer]: A
