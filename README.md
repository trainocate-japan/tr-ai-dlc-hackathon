# ハマリーニング

> ハマる + ラーニング。没入して夢中になっているうちに、気がつけば学びが身についている。

IT/ビジネス研修コースをTikTok風の縦無限スクロールで「時間を溶かしながら」ブラウズし、検索やレコメンドでは得られない偶然の出会い（セレンディピティ）を通じて学びを提供するWebアプリケーションです。

---

## ハッカソン情報

| 項目 | 内容 |
|------|------|
| **イベント** | [AWS Summit Japan 2026 ハッカソン](https://pages.awscloud.com/rs/112-TZM-766/images/AWS-Summit-Japan-2026-Hackathon_%E8%AA%AC%E6%98%8E%E4%BC%9A_%E9%85%8D%E5%B8%83%E7%94%A8.pdf) |
| **テーマ** | AI-DLC（AI-Driven Development Life Cycle） |
| **開発環境** | [Kiro](https://kiro.dev/) — AI-powered IDE |
| **ワークフロー** | AI-DLC アダプティブワークフロー（Inception → Construction → Operations） |

本プロジェクトは、AWS Summit Japan 2026 ハッカソンへの提出作品です。AI-DLC（AI駆動開発ライフサイクル）のワークフローに従い、Kiro開発環境を活用して設計・実装を行っています。

---

## コンセプト

- 🔄 **検索しない** — 目的なく眺めて、偶然の出会いを楽しむ
- 🕳️ **没入する** — TikTok風UIで時間が溶ける体験
- 🧠 **気がつけば学んでいる** — コレクションと発見マップで学びを可視化
- ❓ **○×クイズ** — フィード内にクイズカードが混在し、答えるとポイント＆レベルアップ

## 主な機能

| 機能 | 説明 |
|------|------|
| **無限スクロールフィード** | コース情報カードとクイズカードがTikTok風に流れる |
| **コレクション** | いいね・あとで見るでコースを保存 |
| **発見マップ** | 閲覧コースをグラフ状にビジュアライズ、カテゴリ関係を線で表示 |
| **○×クイズ＆ポイント** | AI生成クイズに答えてポイント獲得、レベルアップ |
| **AI生成コンテンツ** | Amazon Bedrockがコース概要（HTML）とクイズを自動生成 |
| **ユーザー認証** | Cognito（メール/パスワード + Googleログイン） |

## アーキテクチャ

イベント駆動型サーバーレスアーキテクチャ（AWS）

```
Frontend (Next.js)          API Gateway (Edge Optimized)
    |                              |
    v                              v
CloudFront + S3             Lambda (Python) × N
                                   |
                                   v
                              DynamoDB (6 tables)

Batch: S3 → EventBridge → SQS → Lambda → DynamoDB
AI:    DynamoDB Streams → Lambda → EventBridge → SQS → Lambda + Bedrock
```

## 技術スタック

| レイヤー | 技術 |
|----------|------|
| フロントエンド | Next.js (App Router), React, TypeScript |
| ホスティング | S3 + CloudFront |
| API | API Gateway (Edge Optimized) + Lambda (Python) |
| 認証 | Amazon Cognito |
| データベース | DynamoDB |
| AI/コンテンツ生成 | Amazon Bedrock |
| IaC | AWS CDK (TypeScript) |
| 監視 | ADOT Lambda Layer + X-Ray |

## プロジェクト構成（モノレポ）

```
hamlearning/
├── infrastructure/     # AWS CDK (6スタック)
├── backend/            # Lambda ハンドラー (API)
├── batch/              # Lambda ハンドラー (バッチ/AI生成)
├── frontend/           # Next.js アプリ
├── docs/               # API仕様等
└── aidlc-docs/         # 設計ドキュメント
```

## 開発順序

1. **Infrastructure** — CDKで全AWSリソースをデプロイ
2. **Backend API + Batch Processing** — 並行開発可能
3. **Frontend** — APIが稼働してから本格開発

## データフロー

### コースデータ登録
```
社内システム → S3バケット → EventBridge → SQS → Lambda → DynamoDB (courses)
```

### AI コンテンツ生成
```
DynamoDB Streams (courses) → Lambda → EventBridge
  → SQS (summary) → Lambda + Bedrock → DynamoDB (course-summaries) [HTML形式]
  → SQS (quiz)    → Lambda + Bedrock → DynamoDB (quizzes) [○×形式]
```

### ユーザー体験
```
ユーザー → フィード閲覧 → いいね/あとで見る → コレクション → 発見マップ
                        → クイズ回答 → ポイント加算 → レベルアップ
```

## 対象ユーザー

社会人全般（IT/ビジネススキルに興味がある方）

## 目的

トレノケートの自社サービスサイトへの流入を目的とした、研修コース発見プラットフォーム。

## ドキュメント

詳細な設計ドキュメントは `aidlc-docs/` を参照してください。

- [要件定義](aidlc-docs/inception/requirements/requirements.md)
- [ユーザーストーリー](aidlc-docs/inception/user-stories/stories.md)
- [アプリケーション設計](aidlc-docs/inception/application-design/application-design.md)
- [ユニット分解](aidlc-docs/inception/application-design/unit-of-work.md)

## ライセンス

TBD
