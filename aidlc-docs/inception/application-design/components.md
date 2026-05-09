# コンポーネント定義

## フロントエンドコンポーネント

### FE-01: DashboardPage
**責務**: ユーザーのポイント状況・達成チェックポイントを可視化するメインページ
- 累計ポイント・今日のポイント・週間ポイントの表示
- 最近の達成チェックポイント一覧
- ポイント推移グラフ
- レベル・バッジ表示
- SSEによるリアルタイムポイント更新受信

### FE-02: ChatPage
**責務**: AIとの対話窓口（ユーザーとシステムの唯一のインタラクション画面）
- 行動後の振り返り入力
- 行動前の予定入力
- AIからの応答表示（同期・非同期ACK両対応）
- 会話履歴表示

### FE-03: SettingsPage
**責務**: データソース連携設定・ユーザー設定
- データソース連携一覧（GitHub・Google Calendar・Slack・Google Meet）
- OAuth認可フロー起動
- 連携状態表示・解除

### FE-04: AdminPage（管理者専用フロントエンド）
**責務**: 管理者向け操作画面（同一バックエンドAPIを共有、フロントエンドのみ分離）
- チェックポイント一覧・レビュー・削除
- ユーザー一覧・緊急削除

### FE-05: PointDisplay（共有コンポーネント）
**責務**: ポイント数値の表示（DashboardPage・ChatPage で再利用）

### FE-06: CheckpointCard（共有コンポーネント）
**責務**: 達成済みチェックポイントの1件表示カード

### FE-07: DataSourceConnector（共有コンポーネント）
**責務**: 各データソースの連携状態表示・OAuth起動ボタン

---

## バックエンドコンポーネント

### BE-01: UserHandler
**責務**: ユーザー識別・セッション管理
- メールアドレスによる簡易識別（Phase 1）
- Cognito認証への移行対応（Phase 1.5）

### BE-02: CheckpointHandler
**責務**: チェックポイントのCRUD操作（管理者・システムのみ）
- チェックポイント登録・更新・削除
- 達成済みチェックポイント取得（ユーザー向け）

### BE-03: PointHandler
**責務**: ポイントの集計・履歴管理
- ポイント加算
- 累計・期間別集計
- ポイント履歴取得

### BE-04: ChatHandler
**責務**: AIチャットリクエストの受付・応答
- 同期応答（短時間処理）
- 即時ACK + 非同期処理（長時間処理）
- 会話履歴管理

### BE-05: DataSourceWebhookHandler
**責務**: 外部データソースからのWebhookイベント受信
- GitHub Webhook受信
- Slack Events API受信
- Google Calendar Push通知受信

### BE-06: AchievementJudgeHandler
**責務**: チェックポイント達成判定のトリガー管理
- データソースイベント受信後に判定キューへ投入
- 判定結果のポイント反映

### BE-07: AdminHandler
**責務**: 管理者操作API
- チェックポイント管理（一覧・削除）
- ユーザー管理（一覧・削除）
- 監査ログ記録

### BE-08: SSEHandler
**責務**: Server-Sent Events によるリアルタイム更新配信
- ポイント更新イベント配信
- 達成通知配信（Phase 1.4）

---

## AI/LLMコンポーネント

### AI-01: LLMProvider（抽象インターフェース）
**責務**: LLMバックエンドの抽象化レイヤー
- ローカルLLM（Ollama）とAmazon Bedrockを差し替え可能
- 環境変数（LLM_BACKEND=local|bedrock）で選択

### AI-02: LocalLLMAdapter
**責務**: Ollama（ローカルLLM）への接続実装
- 開発環境・プライバシー配慮が必要な処理で使用

### AI-03: BedrockAdapter
**責務**: Amazon Bedrock への接続実装
- 本番環境・高精度推論・大量処理で使用

### AI-04: CheckpointGenerator
**責務**: データソースのイベントデータからチェックポイントを生成
- LLMProviderを通じてLLMに処理を委譲
- 非同期処理

### AI-05: ChatResponder
**責務**: AIチャットの応答生成
- LLMProviderを通じてLLMに処理を委譲
- 短時間: 同期応答 / 長時間: 即時ACK + 非同期完了通知

---

## データソース連携コンポーネント

### DS-01: OAuthManager
**責務**: 各データソースのOAuth 2.0認可フロー管理
- 認可URL生成・コールバック処理・トークン保存

### DS-02: GitHubConnector
**責務**: GitHub Webhook受信・イベント正規化

### DS-03: GoogleCalendarConnector
**責務**: Google Calendar Push通知受信・イベント正規化
- Google Meet連携（カレンダーイベントからミーティング検出）

### DS-04: SlackConnector
**責務**: Slack Events API受信・イベント正規化

### DS-05: DataSourceEventNormalizer
**責務**: 各ソースの異なるイベント形式を統一フォーマットに変換
- CheckpointGeneratorへの入力データを標準化

---

## インフラコンポーネント

### INF-01: APIGateway
**責務**: REST API + SSEエンドポイントの管理・ルーティング

### INF-02: MessageQueue（SQS等）
**責務**: 非同期処理キュー（チェックポイント生成・達成判定）

### INF-03: Database（DynamoDB等）
**責務**: チェックポイント・ポイント履歴・ユーザー・連携トークンの永続化

### INF-04: DockerCompose（開発環境）
**責務**: ローカル開発環境の統合起動
- フロントエンド・バックエンド・Ollama・LocalStack
