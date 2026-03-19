---
layout: default
title: 統合業務申請・承認プラットフォーム 設計ドキュメント
---

# 統合業務申請・承認プラットフォーム

MVP（経費精算モジュール）の設計ドキュメントです。

## プロジェクト概要

| 項目 | 内容 |
|------|------|
| システム名 | 統合業務申請・承認プラットフォーム |
| スコープ | MVP — 経費精算モジュール |
| フロントエンド | React + TypeScript |
| バックエンド | Python + FastAPI |
| インフラ | AWS サーバーレス（Lambda + API Gateway） |
| データベース | PostgreSQL（RDS） |

## ドキュメント一覧

### 要件定義

- [要件定義書](aidlc-docs/inception/requirements/requirements.md)
- [要件検証質問](aidlc-docs/inception/requirements/requirement-verification-questions.md)

### アプリケーション設計

- [アプリケーション設計書（統合版）](aidlc-docs/inception/application-design/application-design.md)
- [コンポーネント定義](aidlc-docs/inception/application-design/components.md)
- [サービス定義](aidlc-docs/inception/application-design/services.md)
- [コンポーネントメソッド定義](aidlc-docs/inception/application-design/component-methods.md)
- [コンポーネント依存関係](aidlc-docs/inception/application-design/component-dependency.md)

### 計画

- [実行計画](aidlc-docs/inception/plans/execution-plan.md)
- [アプリケーション設計計画](aidlc-docs/inception/plans/application-design-plan.md)

### プロジェクト状態

- [AI-DLC 状態追跡](aidlc-docs/aidlc-state.md)
