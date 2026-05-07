# サービス定義

## サービスアーキテクチャ概要

「ハマリーニング」はイベント駆動型サーバーレスアーキテクチャを採用。
API層とバッチ処理層を明確に分離し、非同期処理でスケーラビリティを確保する。

---

## 1. API サービス層

### 1.1 認証サービス（Cognito）
| 項目 | 内容 |
|------|------|
| **責務** | ユーザー登録、ログイン、トークン発行・検証 |
| **パターン** | Cognito User Pool + Google Identity Provider |
| **連携** | API Gateway Cognito Authorizer |

### 1.2 フィードオーケストレーション
| 項目 | 内容 |
|------|------|
| **責務** | コース情報カードとクイズカードを混在させたフィードの構築 |
| **パターン** | 単一Lambda内でDynamoDB複数テーブル（courses, course-summaries, quizzes, view-history）を参照し、多様性アルゴリズムを適用 |
| **データフロー** | リクエスト → 閲覧済み除外 → カテゴリ多様性選定 → カード混在 → レスポンス |

### 1.3 コレクションオーケストレーション
| 項目 | 内容 |
|------|------|
| **責務** | いいね/あとで見るのCRUD操作 |
| **パターン** | 各操作が独立したLambda。collectionsテーブルに対するシンプルなCRUD |

### 1.4 クイズ回答オーケストレーション
| 項目 | 内容 |
|------|------|
| **責務** | クイズ回答の正誤判定、ポイント加算、レベル計算 |
| **パターン** | 回答Lambda内でquizzesテーブルから正解取得 → 判定 → usersテーブルのポイント更新（条件付き書き込み） |

### 1.5 ディスカバリーオーケストレーション
| 項目 | 内容 |
|------|------|
| **責務** | 閲覧履歴の集計、カテゴリ別統計の算出 |
| **パターン** | view-historyテーブルをスキャン/クエリし、カテゴリ別に集計 |

---

## 2. バッチ処理サービス層

### 2.1 データ取り込みパイプライン
```
S3 (ファイル配置)
  → EventBridge (S3イベント通知)
    → SQS (course-ingestion-queue / ranking-ingestion-queue)
      → Lambda (process_course_file / process_ranking_file)
        → DynamoDB (courses テーブル)
```

| 項目 | 内容 |
|------|------|
| **責務** | コースデータ/ランキングデータの取り込みと永続化 |
| **パターン** | イベント駆動、SQSによるバッファリングとリトライ |
| **エラーハンドリング** | DLQ（Dead Letter Queue）でエラーメッセージを退避 |

### 2.2 コンテンツ生成パイプライン
```
DynamoDB Streams (courses テーブル変更)
  → Lambda (stream_handler)
    → EventBridge (カスタムイベント)
      → SQS (summary-generation-queue)
        → Lambda (generate_summary) → DynamoDB (course-summaries)
      → SQS (quiz-generation-queue)
        → Lambda (generate_quiz) → DynamoDB (quizzes)
```

| 項目 | 内容 |
|------|------|
| **責務** | コースデータ変更をトリガーに、Bedrockで概要サマリーとクイズを自動生成 |
| **パターン** | イベント駆動、ファンアウト（1イベント → 2つの処理パス） |
| **再帰防止** | 生成結果は別テーブル（course-summaries, quizzes）に書き込み、coursesテーブルのStreamsは再トリガーされない |
| **エラーハンドリング** | DLQでBedrock呼び出し失敗を退避、リトライ可能 |

---

## 3. サービス間通信パターン

| パターン | 用途 |
|----------|------|
| **同期（API Gateway → Lambda）** | ユーザー向けAPI全般 |
| **非同期（EventBridge → SQS → Lambda）** | バッチ処理、コンテンツ生成 |
| **ストリーム（DynamoDB Streams → Lambda）** | データ変更検知 |

---

## 4. 横断的関心事

### 4.1 認証・認可
- API Gateway Cognito Authorizer で全APIエンドポイントを保護
- Lambda内でユーザーIDをJWTから取得し、データアクセス制御

### 4.2 エラーハンドリング
- API: 統一エラーレスポンス形式（statusCode, errorCode, message）
- バッチ: DLQによるエラー退避、CloudWatch Alarmsで通知

### 4.3 ロギング
- 構造化ログ（JSON形式）
- リクエストID/相関IDの伝播
- CloudWatch Logsに集約

### 4.4 レート制限
- API Gateway のスロットリング設定
- ユーザー単位のレート制限
