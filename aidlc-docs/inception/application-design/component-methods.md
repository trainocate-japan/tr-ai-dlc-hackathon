# コンポーネントメソッド定義

## 1. Feed Service

### GET /feed
| 項目 | 内容 |
|------|------|
| **目的** | フィードのコースカード＋クイズカードを取得 |
| **入力** | `cursor: str (optional)`, `limit: int (default=10)` |
| **出力** | `{ items: FeedItem[], nextCursor: str }` |
| **認証** | 必須（Cognito JWT） |
| **備考** | カテゴリ多様性保証アルゴリズムをサーバーサイドで実行。閲覧済みコースを除外。情報カードとクイズカードを混在。 |

### POST /feed/view
| 項目 | 内容 |
|------|------|
| **目的** | コース閲覧を記録（3秒以上表示） |
| **入力** | `{ courseCode: str, viewDurationMs: int }` |
| **出力** | `{ success: bool }` |
| **認証** | 必須 |

---

## 2. Course Service

### GET /courses/{courseCode}
| 項目 | 内容 |
|------|------|
| **目的** | コース詳細情報を取得 |
| **入力** | `courseCode: str (path param)` |
| **出力** | `{ courseCode, title, fullDescription, objectives, category, level, duration, targetAudience, relatedSkills, ranking }` |
| **認証** | 必須 |

---

## 3. Collection Service

### POST /collections/likes
| 項目 | 内容 |
|------|------|
| **目的** | コースにいいねを追加 |
| **入力** | `{ courseCode: str }` |
| **出力** | `{ success: bool }` |
| **認証** | 必須 |

### DELETE /collections/likes/{courseCode}
| 項目 | 内容 |
|------|------|
| **目的** | いいねを解除 |
| **入力** | `courseCode: str (path param)` |
| **出力** | `{ success: bool }` |
| **認証** | 必須 |

### POST /collections/watchlater
| 項目 | 内容 |
|------|------|
| **目的** | あとで見るに追加 |
| **入力** | `{ courseCode: str }` |
| **出力** | `{ success: bool }` |
| **認証** | 必須 |

### DELETE /collections/watchlater/{courseCode}
| 項目 | 内容 |
|------|------|
| **目的** | あとで見るを解除（見終わった） |
| **入力** | `courseCode: str (path param)` |
| **出力** | `{ success: bool }` |
| **認証** | 必須 |

### GET /collections/likes
| 項目 | 内容 |
|------|------|
| **目的** | いいね一覧取得 |
| **入力** | `category: str (optional, query param)` |
| **出力** | `{ items: CourseSummary[] }` |
| **認証** | 必須 |

### GET /collections/watchlater
| 項目 | 内容 |
|------|------|
| **目的** | あとで見る一覧取得 |
| **入力** | `category: str (optional, query param)` |
| **出力** | `{ items: CourseSummary[] }` |
| **認証** | 必須 |

---

## 4. Quiz Service

### POST /quizzes/{quizId}/answer
| 項目 | 内容 |
|------|------|
| **目的** | クイズに回答し、正解ならポイント加算 |
| **入力** | `{ selectedOption: str }` |
| **出力** | `{ correct: bool, correctAnswer: str, pointsEarned: int, totalPoints: int, level: int }` |
| **認証** | 必須 |

### GET /users/me/points
| 項目 | 内容 |
|------|------|
| **目的** | 現在のポイントとレベルを取得 |
| **入力** | なし |
| **出力** | `{ totalPoints: int, level: int, nextLevelPoints: int }` |
| **認証** | 必須 |

---

## 5. Discovery Service

### GET /discovery/map
| 項目 | 内容 |
|------|------|
| **目的** | 発見マップ（コースグラフ + 統計）を取得 |
| **入力** | なし |
| **出力** | `{ nodes: { courseCode: str, title: str, summary: str, category: str, liked: bool, viewed: bool }[], edges: { source: str, target: str, relation: str }[], stats: { totalViewed: int, totalLiked: int, totalCategories: int } }` |
| **認証** | 必須 |
| **備考** | ノード=閲覧/いいね済みコース。エッジ=同カテゴリ関係。liked=trueのノードは強調表示用。summaryはコースカードを思い出すための概要。 |

---

## 6. Batch: Course Data Ingestion

### process_course_file
| 項目 | 内容 |
|------|------|
| **目的** | S3のコースデータファイルを解析しDynamoDBに登録/更新 |
| **入力** | SQSメッセージ（S3オブジェクトキー） |
| **出力** | 処理結果ログ（登録数、更新数、エラー数） |
| **ロジック** | コースコードが既存→更新、新規→追加。DynamoDB coursesテーブルに書き込み。 |

---

## 7. Batch: Ranking Data Ingestion

### process_ranking_file
| 項目 | 内容 |
|------|------|
| **目的** | S3のランキングデータファイルを解析しコースに順位を紐づけ |
| **入力** | SQSメッセージ（S3オブジェクトキー） |
| **出力** | 処理結果ログ |
| **ロジック** | coursesテーブルのranking属性を更新 |

---

## 8. Batch: Summary Generation (Bedrock)

### generate_summary
| 項目 | 内容 |
|------|------|
| **目的** | コース全文からカード表示用概要サマリーを生成 |
| **入力** | SQSメッセージ（courseCode） |
| **出力** | course-summariesテーブルに保存 |
| **ロジック** | Bedrock InvokeModel → コース全文をプロンプトに含め、キャッチーな概要サマリーを生成 |

---

## 9. Batch: Quiz Generation (Bedrock)

### generate_quiz
| 項目 | 内容 |
|------|------|
| **目的** | コース全文からクイズを生成 |
| **入力** | SQSメッセージ（courseCode） |
| **出力** | quizzesテーブルに保存 |
| **ロジック** | Bedrock InvokeModel → コース全文をプロンプトに含め、4択クイズ（問題文+選択肢+正解+解説）を生成 |

---

## 10. DynamoDB Streams Handler

### stream_handler
| 項目 | 内容 |
|------|------|
| **目的** | coursesテーブルの変更を検知し、EventBridgeにイベント発行 |
| **入力** | DynamoDB Streamsイベント（INSERT/MODIFY） |
| **出力** | EventBridgeにカスタムイベント発行 |
| **ロジック** | 新規登録/更新されたコースのcourseCodeをイベントペイロードに含め、EventBridgeに送信。EventBridgeルールで2つのSQS（summary-generation-queue, quiz-generation-queue）にルーティング。 |
