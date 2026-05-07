# コンポーネント依存関係

## 依存関係マトリクス

| コンポーネント | 依存先 |
|---------------|--------|
| **Frontend (Next.js)** | API Gateway, Cognito, CloudFront |
| **API Gateway** | Lambda (各API), Cognito (Authorizer) |
| **Feed Lambda** | DynamoDB (courses, course-summaries, quizzes, view-history) |
| **Course Lambda** | DynamoDB (courses) |
| **Collection Lambdas** | DynamoDB (collections) |
| **Quiz Lambda** | DynamoDB (quizzes, users) |
| **Discovery Lambda** | DynamoDB (view-history, collections) |
| **Course Ingestion Lambda** | SQS (course-ingestion-queue), DynamoDB (courses) |
| **Ranking Ingestion Lambda** | SQS (ranking-ingestion-queue), DynamoDB (courses) |
| **Stream Handler Lambda** | DynamoDB Streams (courses), EventBridge |
| **Summary Generation Lambda** | SQS (summary-generation-queue), Bedrock, DynamoDB (courses, course-summaries) |
| **Quiz Generation Lambda** | SQS (quiz-generation-queue), Bedrock, DynamoDB (courses, quizzes) |

---

## データフロー図

### ユーザー向けAPI フロー
```
+----------+     +------------+     +---------+     +----------+
| Frontend | --> | CloudFront | --> | API GW  | --> | Lambda   |
| (Next.js)|     | (CDN)      |     | +Cognito|     | (各API)  |
+----------+     +------------+     +---------+     +----------+
                                                         |
                                                         v
                                                    +----------+
                                                    | DynamoDB  |
                                                    | (各Table) |
                                                    +----------+
```

### バッチ処理（データ取り込み）フロー
```
+---------+     +-------------+     +-----+     +--------+     +----------+
| S3      | --> | EventBridge | --> | SQS | --> | Lambda | --> | DynamoDB |
| (data)  |     | (S3 event)  |     |     |     | (batch)|     | (courses)|
+---------+     +-------------+     +-----+     +--------+     +----------+
```

### コンテンツ生成（Bedrock）フロー
```
+----------+     +----------+     +--------+     +-------------+
| DynamoDB | --> | Lambda   | --> | Event  | --> | EventBridge |
| Streams  |     | (stream) |     | Bridge |     | (rules)     |
| (courses)|     |          |     |        |     |             |
+----------+     +----------+     +--------+     +------+------+
                                                         |
                                        +----------------+----------------+
                                        |                                 |
                                        v                                 v
                                  +-----+------+                   +------+-----+
                                  | SQS        |                   | SQS        |
                                  | (summary)  |                   | (quiz)     |
                                  +-----+------+                   +------+-----+
                                        |                                 |
                                        v                                 v
                                  +-----+------+                   +------+-----+
                                  | Lambda     |                   | Lambda     |
                                  | (summary)  |                   | (quiz gen) |
                                  +-----+------+                   +------+-----+
                                        |                                 |
                                        v                                 v
                                  +-----+------+                   +------+-----+
                                  | Bedrock    |                   | Bedrock    |
                                  +-----+------+                   +------+-----+
                                        |                                 |
                                        v                                 v
                                  +-----+------+                   +------+-----+
                                  | DynamoDB   |                   | DynamoDB   |
                                  | (summaries)|                   | (quizzes)  |
                                  +------------+                   +------------+
```

---

## イベントフロー一覧

| トリガー | イベント | ターゲット | 結果 |
|----------|---------|-----------|------|
| S3 PutObject (course data) | EventBridge S3 event | course-ingestion-queue | コースデータ登録/更新 |
| S3 PutObject (ranking data) | EventBridge S3 event | ranking-ingestion-queue | ランキング更新 |
| DynamoDB courses INSERT/MODIFY | DynamoDB Streams | stream_handler Lambda | EventBridgeイベント発行 |
| EventBridge custom event | course.updated | summary-generation-queue | 概要サマリー生成 |
| EventBridge custom event | course.updated | quiz-generation-queue | クイズ生成 |

---

## テーブル間の関係

```
+------------+          +------------------+
| courses    | -------> | course-summaries |
| (master)   |  1:1     | (AI generated)   |
+------------+          +------------------+
      |
      | 1:N
      v
+------------+
| quizzes    |
| (AI gen)   |
+------------+

+------------+          +------------------+
| users      | -------> | collections      |
| (profile)  |  1:N     | (likes/watchlater)|
+------------+          +------------------+
      |
      | 1:N
      v
+------------+
| view-history|
+------------+
```
