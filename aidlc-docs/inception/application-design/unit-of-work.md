# ユニット定義

## ユニット構成（4ユニット）

| # | ユニット名 | 責務 | 技術 |
|---|-----------|------|------|
| 1 | **Infrastructure** | 全AWSリソースのIaC定義 | AWS CDK (TypeScript) |
| 2 | **Backend API** | ユーザー向けREST API全般 | Python, Lambda |
| 3 | **Batch Processing** | データ取り込み、Bedrockコンテンツ生成 | Python, Lambda, Bedrock |
| 4 | **Frontend** | WebアプリケーションUI | Next.js, React, TypeScript |

---

## Unit 1: Infrastructure

### 責務
- 全AWSリソースのCDK定義（1プロジェクト、スタック分割）
- DynamoDBテーブル、S3バケット、API Gateway、Cognito、SQS、EventBridge、Lambda関数定義
- CloudFront配信設定
- IAMロール・ポリシー
- 監視・アラーム設定

### CDKスタック分割
| スタック | リソース |
|----------|----------|
| **DataStack** | DynamoDB 6テーブル、S3バケット（data, frontend） |
| **AuthStack** | Cognito User Pool、Google IdP設定 |
| **ApiStack** | API Gateway (Edge Optimized)、Lambda関数（API系）、Cognito Authorizer |
| **BatchStack** | EventBridgeルール、SQSキュー4本、DLQ、Lambda関数（バッチ系）、DynamoDB Streams設定 |
| **ObservabilityStack** | X-Ray設定、CloudWatch Alarms、ADOT Lambda Layer |
| **FrontendStack** | CloudFront Distribution、S3バケットポリシー |

### 境界
- アプリケーションコード（Lambda関数のハンドラー）は含まない
- Lambda関数の「定義」（メモリ、タイムアウト、環境変数、レイヤー）のみ
- 実際のハンドラーコードはBackend API / Batch Processingユニットが提供

---

## Unit 2: Backend API

### 責務
- ユーザー向けREST APIのLambdaハンドラー実装
- Feed Service（フィード取得、閲覧記録）
- Course Service（コース詳細取得）
- Collection Service（いいね、あとで見る CRUD）
- Quiz Service（クイズ回答、ポイント加算）
- Discovery Service（発見マップデータ構築）

### 境界
- API Gatewayの定義はInfrastructureユニット
- DynamoDBテーブルの定義はInfrastructureユニット
- このユニットはLambdaハンドラーのPythonコードのみ

---

## Unit 3: Batch Processing

### 責務
- コースデータ取り込み（S3 → DynamoDB）
- ランキングデータ取り込み（S3 → DynamoDB）
- DynamoDB Streams Handler（変更検知 → EventBridge発行）
- 概要サマリー生成（Bedrock呼び出し → course-summariesテーブル）
- クイズ生成（Bedrock呼び出し → quizzesテーブル）

### 境界
- SQS/EventBridge/DynamoDB Streamsの定義はInfrastructureユニット
- このユニットはLambdaハンドラーのPythonコードのみ
- Bedrockのプロンプト設計もこのユニットに含む

---

## Unit 4: Frontend

### 責務
- Next.js App RouterによるWebアプリケーション
- TikTok風無限スクロールフィード
- コレクション管理画面
- 発見マップ（グラフビジュアライゼーション）
- クイズ回答UI
- ユーザー認証UI（Cognito連携）
- ポイント・レベル表示

### 境界
- S3/CloudFrontの定義はInfrastructureユニット
- API通信はBackend APIユニットが提供するエンドポイントを利用

---

## コード構成（モノレポ）

```
hamlearning/
├── infrastructure/          # Unit 1: CDK Project
│   ├── bin/
│   │   └── app.ts
│   ├── lib/
│   │   ├── data-stack.ts
│   │   ├── auth-stack.ts
│   │   ├── api-stack.ts
│   │   ├── batch-stack.ts
│   │   ├── observability-stack.ts
│   │   └── frontend-stack.ts
│   ├── cdk.json
│   ├── package.json
│   └── tsconfig.json
│
├── backend/                 # Unit 2: Backend API
│   ├── src/
│   │   ├── feed/
│   │   │   ├── get_feed.py
│   │   │   └── post_view.py
│   │   ├── courses/
│   │   │   └── get_course.py
│   │   ├── collections/
│   │   │   ├── post_like.py
│   │   │   ├── delete_like.py
│   │   │   ├── post_watchlater.py
│   │   │   ├── delete_watchlater.py
│   │   │   ├── get_likes.py
│   │   │   └── get_watchlater.py
│   │   ├── quizzes/
│   │   │   ├── post_answer.py
│   │   │   └── get_points.py
│   │   ├── discovery/
│   │   │   └── get_map.py
│   │   └── shared/
│   │       ├── models.py
│   │       ├── dynamodb.py
│   │       └── auth.py
│   ├── tests/
│   ├── requirements.txt
│   └── pyproject.toml
│
├── batch/                   # Unit 3: Batch Processing
│   ├── src/
│   │   ├── ingestion/
│   │   │   ├── process_course_file.py
│   │   │   └── process_ranking_file.py
│   │   ├── streams/
│   │   │   └── stream_handler.py
│   │   ├── generation/
│   │   │   ├── generate_summary.py
│   │   │   ├── generate_quiz.py
│   │   │   └── prompts/
│   │   │       ├── summary_prompt.py
│   │   │       └── quiz_prompt.py
│   │   └── shared/
│   │       ├── models.py
│   │       ├── dynamodb.py
│   │       └── bedrock.py
│   ├── tests/
│   ├── requirements.txt
│   └── pyproject.toml
│
├── frontend/                # Unit 4: Frontend
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx
│   │   │   ├── feed/
│   │   │   ├── collections/
│   │   │   ├── discovery/
│   │   │   ├── profile/
│   │   │   └── auth/
│   │   ├── components/
│   │   │   ├── feed/
│   │   │   ├── quiz/
│   │   │   ├── collection/
│   │   │   ├── discovery/
│   │   │   └── common/
│   │   ├── hooks/
│   │   ├── lib/
│   │   │   ├── api.ts
│   │   │   └── auth.ts
│   │   └── types/
│   ├── public/
│   ├── next.config.js
│   ├── package.json
│   └── tsconfig.json
│
├── docs/                    # 共有ドキュメント
│   └── api-spec.md
└── README.md
```
