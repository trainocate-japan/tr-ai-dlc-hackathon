# ユニット依存関係

## 依存関係マトリクス

| ユニット | 依存先 | 依存の種類 |
|----------|--------|-----------|
| **Frontend** | Backend API | API呼び出し（HTTP） |
| **Frontend** | Infrastructure | デプロイ先（S3, CloudFront） |
| **Backend API** | Infrastructure | 実行環境（Lambda, API GW, DynamoDB） |
| **Batch Processing** | Infrastructure | 実行環境（Lambda, SQS, EventBridge, DynamoDB, Bedrock） |
| **Infrastructure** | なし | 他ユニットに依存しない（最上流） |

## 依存関係図

```
+----------------+
| Infrastructure |  ← 最上流（依存なし）
+-------+--------+
        |
        | デプロイ先・実行環境を提供
        |
+-------+-------------------+------------------+
|                           |                  |
v                           v                  v
+-------------+    +-----------------+    +----------+
| Backend API |    | Batch Processing|    | Frontend |
+-------------+    +-----------------+    +----+-----+
        ^                                      |
        |          API呼び出し                  |
        +--------------------------------------+
```

---

## 開発順序（ボトムアップ）

### Phase 1: Infrastructure（最優先）
- 全AWSリソースのCDK定義
- デプロイして実環境を構築
- 他の全ユニットの前提条件

### Phase 2: Backend API + Batch Processing（並行可能）
- Infrastructureデプロイ後に開発開始
- Backend APIとBatch Processingは互いに依存しないため並行開発可能
- 共通のDynamoDBテーブルを使うが、書き込み先が異なる

### Phase 3: Frontend
- Backend APIのエンドポイントが稼働してから本格開発
- ただしモックAPIで先行開発も可能

---

## 並行開発可能性

| 組み合わせ | 並行可能？ | 備考 |
|-----------|-----------|------|
| Infrastructure + Backend API | ❌ | InfraがBackendの前提 |
| Infrastructure + Batch | ❌ | InfraがBatchの前提 |
| Infrastructure + Frontend | ❌ | InfraがFrontendの前提 |
| Backend API + Batch Processing | ✅ | 互いに独立、DynamoDBテーブル共有だが競合なし。ただしBatchが生成するcourse-summaries/quizzesをBackend APIが参照するため、テーブルスキーマの合意が必要 |
| Backend API + Frontend | ⚠️ | モックAPI使えば部分的に並行可能 |
| Batch Processing + Frontend | ✅ | 直接の依存なし |

---

## インターフェース契約

| インターフェース | 提供元 | 利用元 | 定義場所 |
|----------------|--------|--------|----------|
| REST API仕様 | Backend API | Frontend | docs/api-spec.md |
| DynamoDBテーブルスキーマ | Infrastructure | Backend API, Batch | infrastructure/lib/data-stack.ts |
| SQSメッセージ形式 | Infrastructure | Batch Processing | infrastructure/lib/batch-stack.ts |
| Cognito設定 | Infrastructure | Frontend, Backend API | infrastructure/lib/auth-stack.ts |
