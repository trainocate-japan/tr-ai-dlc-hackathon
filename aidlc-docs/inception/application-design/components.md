# コンポーネント定義

## 1. フロントエンド

### 1.1 Web Application (Next.js App Router)
| 項目 | 内容 |
|------|------|
| **技術** | Next.js 14+ (App Router), React, TypeScript |
| **ホスティング** | S3 + CloudFront (Static Export) |
| **責務** | ユーザーインターフェース、クライアントサイド状態管理、API通信 |

**主要ページ/コンポーネント:**
- フィードページ（無限スクロール、コースカード、クイズカード）
- コレクションページ（いいね一覧、あとで見る一覧）
- 発見マップページ（カテゴリ別閲覧可視化）
- プロフィールページ（ポイント、レベル、設定）
- 認証ページ（ログイン、登録）

---

## 2. バックエンドAPI（Lambda関数群）

### 2.1 Feed Service
| 項目 | 内容 |
|------|------|
| **技術** | Python, Lambda |
| **責務** | フィード取得、コース選定アルゴリズム（多様性保証）、閲覧履歴記録 |

**Lambda関数:**
- `GET /feed` — フィード取得（コース情報カード + クイズカードを混在）
- `POST /feed/view` — 閲覧記録（3秒以上表示されたコースを記録）

### 2.2 Course Service
| 項目 | 内容 |
|------|------|
| **技術** | Python, Lambda |
| **責務** | コース詳細取得 |

**Lambda関数:**
- `GET /courses/{courseCode}` — コース詳細取得

### 2.3 Collection Service
| 項目 | 内容 |
|------|------|
| **技術** | Python, Lambda |
| **責務** | いいね管理、あとで見る管理、コレクション一覧取得 |

**Lambda関数:**
- `POST /collections/likes` — いいね追加
- `DELETE /collections/likes/{courseCode}` — いいね解除
- `POST /collections/watchlater` — あとで見る追加
- `DELETE /collections/watchlater/{courseCode}` — あとで見る解除
- `GET /collections/likes` — いいね一覧取得
- `GET /collections/watchlater` — あとで見る一覧取得

### 2.4 Quiz Service
| 項目 | 内容 |
|------|------|
| **技術** | Python, Lambda |
| **責務** | クイズ回答処理、ポイント加算、レベル計算 |

**Lambda関数:**
- `POST /quizzes/{quizId}/answer` — クイズ回答＆ポイント加算
- `GET /users/me/points` — ポイント・レベル取得

### 2.5 Discovery Service
| 項目 | 内容 |
|------|------|
| **技術** | Python, Lambda |
| **責務** | 発見マップデータ取得、閲覧統計 |

**Lambda関数:**
- `GET /discovery/map` — 発見マップ（カテゴリ別閲覧統計）取得

---

## 3. バッチ処理

### 3.1 Course Data Ingestion
| 項目 | 内容 |
|------|------|
| **技術** | Python, Lambda |
| **トリガー** | S3 PutObject → EventBridge → SQS → Lambda |
| **責務** | コースデータファイル解析、DynamoDB登録/更新（コースコードで判定） |

### 3.2 Ranking Data Ingestion
| 項目 | 内容 |
|------|------|
| **技術** | Python, Lambda |
| **トリガー** | S3 PutObject → EventBridge → SQS → Lambda |
| **責務** | 月間ランキングデータ取り込み、コースへの順位紐づけ |

### 3.3 Summary Generation (Bedrock)
| 項目 | 内容 |
|------|------|
| **技術** | Python, Lambda, Amazon Bedrock (Amazon Nova 2 Lite) |
| **トリガー** | DynamoDB Streams (courses) → Lambda → EventBridge → SQS → Lambda |
| **責務** | コース全文からカード表示用概要サマリーをHTML形式で生成、course-summariesテーブルに保存 |

### 3.4 Quiz Generation (Bedrock)
| 項目 | 内容 |
|------|------|
| **技術** | Python, Lambda, Amazon Bedrock (Amazon Nova 2 Lite) |
| **トリガー** | DynamoDB Streams (courses) → Lambda → EventBridge → SQS → Lambda |
| **責務** | コース全文から○×2択クイズ（問題文+正解+解説）を生成、quizzesテーブルに保存 |

---

## 4. データストア

### 4.1 DynamoDB Tables
| テーブル | 用途 |
|----------|------|
| **courses** | コースマスターデータ（全文含む、ランキング情報） |
| **course-summaries** | Bedrock生成の概要サマリー（カード表示用） |
| **quizzes** | Bedrock生成のクイズデータ |
| **users** | ユーザー情報（ポイント、レベル） |
| **collections** | いいね、あとで見る |
| **view-history** | 閲覧履歴 |

### 4.2 S3 Buckets
| バケット | 用途 |
|----------|------|
| **hamlearning-data** | コースデータファイル、ランキングデータファイル（管理者配置） |
| **hamlearning-frontend** | Next.js静的ビルド成果物 |

### 4.3 Amazon Cognito
| 項目 | 内容 |
|------|------|
| **User Pool** | ユーザー認証（メール/パスワード + Google連携） |
| **責務** | 登録、ログイン、トークン発行、セッション管理 |

---

## 5. インフラ/ネットワーク

### 5.1 CloudFront
- フロントエンド配信（S3オリジン）のみ
- HTTPSエンドポイント
- セキュリティヘッダー付与

### 5.2 API Gateway (REST API - Edge Optimized)
- バックエンドAPIのエントリーポイント
- エッジ最適化型（CloudFront不要、API GW自体がエッジロケーション活用）
- Cognito Authorizer による認証
- リクエストバリデーション
- レート制限

### 5.3 EventBridge
- S3イベントのルーティング
- DynamoDB Streams後のイベント分配

### 5.4 SQS Queues
| キュー | 用途 |
|--------|------|
| **course-ingestion-queue** | コースデータ取り込みジョブ |
| **ranking-ingestion-queue** | ランキングデータ取り込みジョブ |
| **summary-generation-queue** | 概要サマリー生成ジョブ |
| **quiz-generation-queue** | クイズ生成ジョブ |
