# ストーリー → ユニット マッピング

## マッピング一覧

| ストーリー | タイトル | ユニット | 備考 |
|-----------|---------|---------|------|
| US-001 | ユーザー登録 | Infrastructure + Frontend | Cognito設定(Infra) + 登録UI(Frontend) |
| US-002 | ログイン | Infrastructure + Frontend | Cognito設定(Infra) + ログインUI(Frontend) |
| US-003 | コースフィードの無限スクロール | Backend API + Frontend | フィードAPI(Backend) + スクロールUI(Frontend) |
| US-004 | コースカードのビジュアル表示 | Frontend | カードコンポーネント(Frontend) |
| US-005 | フィードの多様性保証 | Backend API | 多様性アルゴリズム(Backend) |
| US-006 | コースに「いいね」する | Backend API + Frontend | いいねAPI(Backend) + ボタンUI(Frontend) |
| US-007 | コースを「あとで見る」に追加 | Backend API + Frontend | あとで見るAPI(Backend) + ボタンUI(Frontend) |
| US-008 | コース詳細の表示 | Backend API + Frontend | 詳細API(Backend) + 詳細画面(Frontend) |
| US-009 | コレクション一覧の表示 | Backend API + Frontend | 一覧API(Backend) + コレクション画面(Frontend) |
| US-010 | コレクションのカテゴリ別表示 | Backend API + Frontend | フィルタAPI(Backend) + フィルタUI(Frontend) |
| US-011 | 閲覧履歴の自動記録 | Backend API + Frontend | 記録API(Backend) + 3秒検知(Frontend) |
| US-012 | 発見マップの表示 | Backend API + Frontend | グラフデータAPI(Backend) + グラフUI(Frontend) |
| US-013 | コースデータの一括登録 | Infrastructure + Batch | S3/SQS設定(Infra) + 取り込みLambda(Batch) |
| US-014 | コースデータの更新 | Batch | 更新ロジック(Batch) |
| US-015 | クイズカードの表示と回答 | Backend API + Frontend | 回答API(Backend) + クイズUI(Frontend) |
| US-016 | ポイント加算とユーザーレベル | Backend API + Frontend | ポイントAPI(Backend) + レベル表示(Frontend) |
| US-017 | クイズ・概要の自動生成 | Infrastructure + Batch | Streams/SQS設定(Infra) + Bedrock Lambda(Batch) |
| US-018 | 月間ランキングデータの取り込み | Infrastructure + Batch | S3/SQS設定(Infra) + 取り込みLambda(Batch) |

---

## ユニット別ストーリーカバレッジ

### Unit 1: Infrastructure
| ストーリー | 関与内容 |
|-----------|---------|
| US-001, US-002 | Cognito User Pool, Google IdP |
| US-013, US-018 | S3バケット, EventBridgeルール, SQSキュー |
| US-017 | DynamoDB Streams, EventBridge, SQS (summary/quiz) |
| 全API系 | API Gateway, Lambda定義, DynamoDBテーブル |

### Unit 2: Backend API
| ストーリー | 関与内容 |
|-----------|---------|
| US-003, US-005 | Feed Service（フィード取得、多様性アルゴリズム） |
| US-006, US-007, US-009, US-010 | Collection Service（CRUD） |
| US-008 | Course Service（詳細取得） |
| US-011 | Feed Service（閲覧記録） |
| US-012 | Discovery Service（グラフデータ構築） |
| US-015, US-016 | Quiz Service（回答、ポイント） |

### Unit 3: Batch Processing
| ストーリー | 関与内容 |
|-----------|---------|
| US-013, US-014 | コースデータ取り込み・更新 |
| US-017 | Bedrock概要サマリー生成、クイズ生成 |
| US-018 | ランキングデータ取り込み |

### Unit 4: Frontend
| ストーリー | 関与内容 |
|-----------|---------|
| US-001, US-002 | 認証UI（登録、ログイン） |
| US-003, US-004 | フィードUI（無限スクロール、カード表示） |
| US-006, US-007 | アクションUI（いいね、あとで見るボタン） |
| US-008 | コース詳細画面 |
| US-009, US-010 | コレクション画面 |
| US-011 | 閲覧時間検知（3秒タイマー） |
| US-012 | 発見マップ（グラフビジュアライゼーション） |
| US-015 | クイズカードUI（○×ボタン） |
| US-016 | ポイント・レベル表示 |

---

## カバレッジ確認

- ✅ 全18ストーリー（US-001〜US-018）が少なくとも1つのユニットに割り当て済み
- ✅ 全4ユニットに少なくとも1つのストーリーが割り当て済み
- ✅ クロスユニットストーリー（複数ユニットにまたがる）が明確に識別済み
