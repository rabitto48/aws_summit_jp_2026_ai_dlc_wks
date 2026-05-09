# コンポーネントメソッド定義

詳細なビジネスロジックは Construction Phase の Functional Design で定義する。
ここではメソッドシグネチャと高レベルの目的を定義する。

---

## BE-01: UserHandler

```
identifyUser(email: string): User
  - メールアドレスからユーザーを識別または新規作成
  - 戻り値: Userオブジェクト（userId, email, createdAt）

getUser(userId: string): User
  - ユーザー情報取得
```

---

## BE-02: CheckpointHandler

```
createCheckpoint(data: CheckpointInput): Checkpoint
  - システム/管理者のみ呼び出し可能
  - チェックポイント登録

deleteCheckpoint(checkpointId: string): void
  - 管理者のみ呼び出し可能
  - 関連ポイントの取り消しも実行

getAchievedCheckpoints(userId: string, options: PaginationOptions): Checkpoint[]
  - 達成済みチェックポイント一覧取得（ユーザー向け）
  - 未達成チェックポイントは返さない

listAllCheckpoints(options: FilterOptions): Checkpoint[]
  - 管理者向け全チェックポイント一覧
```

---

## BE-03: PointHandler

```
addPoints(userId: string, checkpointId: string, points: number): PointRecord
  - ポイント加算・履歴記録

revokePoints(userId: string, checkpointId: string): void
  - チェックポイント削除時のポイント取り消し

getSummary(userId: string): PointSummary
  - 累計・今日・週間ポイントの集計
  - 戻り値: { total, today, weekly }

getHistory(userId: string, options: PaginationOptions): PointRecord[]
  - ポイント履歴取得（時系列）
```

---

## BE-04: ChatHandler

```
sendMessage(userId: string, message: string, type: 'reflection' | 'plan'): ChatResponse
  - type='reflection': 行動後の振り返り入力
  - type='plan': 行動前の予定入力
  - 短時間処理: 同期応答を返す
  - 長時間処理: 即時ACKを返し非同期処理をキューに投入

getHistory(userId: string, options: PaginationOptions): ChatMessage[]
  - 会話履歴取得
```

---

## BE-05: DataSourceWebhookHandler

```
handleGitHubEvent(payload: GitHubWebhookPayload, signature: string): void
  - Webhook署名検証 → DS-02 GitHubConnector に委譲

handleSlackEvent(payload: SlackEventPayload, token: string): void
  - トークン検証 → DS-04 SlackConnector に委譲

handleGoogleCalendarEvent(payload: GoogleCalendarPushPayload): void
  - DS-03 GoogleCalendarConnector に委譲
```

---

## BE-06: AchievementJudgeHandler

```
judgeAchievement(event: NormalizedEvent, userId: string): AchievementResult
  - 正規化イベントをもとに達成判定
  - 達成確定時: PointHandler.addPoints() を呼び出し
  - 戻り値: { achieved: boolean, checkpointId?, points? }
```

---

## BE-07: AdminHandler

```
listCheckpoints(options: FilterOptions): Checkpoint[]
deleteCheckpoint(checkpointId: string, reason: string): void
  - 監査ログ記録必須

listUsers(options: FilterOptions): User[]
deleteUser(userId: string, reason: string): void
  - 関連データ削除・監査ログ記録必須
```

---

## BE-08: SSEHandler

```
subscribe(userId: string): SSEStream
  - ユーザー向けSSEストリームを開始

publish(userId: string, event: SSEEvent): void
  - 特定ユーザーへのイベント配信
  - eventType: 'point_update' | 'achievement' | 'chat_complete'

broadcast(event: SSEEvent): void
  - 全接続ユーザーへの配信（管理者操作通知等）
```

---

## AI-01: LLMProvider（インターフェース）

```
interface LLMProvider {
  complete(prompt: string, options: LLMOptions): Promise<string>
  stream(prompt: string, options: LLMOptions): AsyncIterable<string>
}
```

---

## AI-04: CheckpointGenerator

```
generate(event: NormalizedEvent, context: UserContext): Promise<CheckpointDraft[]>
  - 正規化イベントとユーザーコンテキストからチェックポイント候補を生成
  - LLMProvider.complete() を使用（非同期）
  - 戻り値: CheckpointDraft[]（action, points, sourceEventId）
```

---

## AI-05: ChatResponder

```
respond(message: string, history: ChatMessage[], type: 'reflection' | 'plan'): Promise<ChatResponse>
  - LLMProvider を使用して応答生成
  - type='plan' の場合: 予定内容を構造化してチェックポイント生成トリガー
  - 戻り値: { text, isAsync, jobId? }
```

---

## DS-01: OAuthManager

```
getAuthorizationUrl(source: DataSourceType, userId: string): string
handleCallback(source: DataSourceType, code: string, userId: string): OAuthToken
revokeToken(source: DataSourceType, userId: string): void
getToken(source: DataSourceType, userId: string): OAuthToken | null
```

---

## DS-05: DataSourceEventNormalizer

```
normalize(source: DataSourceType, rawEvent: unknown): NormalizedEvent
  - 各ソース固有のイベント形式を統一フォーマットに変換
  - 戻り値: { source, userId, eventType, timestamp, rawData, metadata }
```
