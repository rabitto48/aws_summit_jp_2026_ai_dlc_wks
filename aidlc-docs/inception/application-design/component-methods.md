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

---

## AI-06: PraiseMessageGenerator【認知の歪み演出】

```
generate(checkpointType: string, points: number, isBonus: boolean, context: UserContext): Promise<string>
  - チェックポイント種別・ポイント量・ボーナス有無・ユーザーコンテキストから称賛メッセージを生成
  - LLMProvider.complete() を使用（LocalLLMAdapter優先）
  - 毎回微妙に異なる表現を生成（飽きさせない）
  - 戻り値: 称賛メッセージ文字列（例: 「状況把握、完璧です！」「優先順位の達人！」）

generateDashboardGreeting(userId: string, streakDays: number): Promise<string>
  - ダッシュボード表示時の挨拶メッセージを生成
  - ストリーク日数に応じてメッセージを変化させる
  - 戻り値: 挨拶メッセージ文字列（例: 「今日も頑張りました！」「X日連続、素晴らしい！」）
```

---

## SVC-08: AddictionEngineService【依存性強化】

```
calculateVariableReward(basePoints: number): VariableRewardResult
  - 可変報酬ポイントを計算する
  - basePoints × (1.0 + random(0.0, 0.5)) を計算
  - 20%の確率でボーナスポイント判定（isBonus=true）
  - 戻り値: { calculatedPoints: number, isBonus: boolean }

updateStreak(userId: string): StreakResult
  - ユーザーのストリーク（連続記録）を更新する
  - 当日初回ポイント獲得時に連続日数をインクリメント
  - 前日未獲得の場合はストリークをリセット
  - 戻り値: { currentStreak: number, longestStreak: number, isNewRecord: boolean }

getStreakStatus(userId: string): StreakStatus
  - 現在のストリーク状態を取得する
  - 当日未獲得かどうかを判定
  - 戻り値: { currentStreak, longestStreak, hasPointsToday: boolean, warningMessage?: string }

checkDailyStreak(): void
  - 毎日0時に実行（スケジューラー起動）
  - 当日未獲得ユーザーに損失回避メッセージをSSE配信（Phase 1.4: プッシュ通知）

applyBroadImpactPenalty(userId: string, basePoints: number): number
  - 設定期間内（デフォルト7日）の broad_impact 行動回数を取得する
  - 回数に応じた累積ペナルティ倍率を適用する（1回:1.0x / 2回:1.5x / 3回以上:2.0x）
  - 戻り値: 倍率適用後のポイント（負の値）

checkBroadImpactConsecutive(userId: string): void
  - 設定期間内の broad_impact 行動回数が閾値（デフォルト3回）以上の場合にSSE警告を配信する
  - 警告内容: 「最近X日間で〇回、大きな変化を起こしています。少し立ち止まりませんか？」
  - Phase 1.4: プッシュ通知にも配信
```

---

## BE-09: RewardRuleHandler【報酬勾配ルール管理】

```
listRules(): RewardRule[]
  - ルール一覧を優先度順で返す（管理者のみ）

createRule(input: RewardRuleInput): RewardRule
  - ルール作成。監査ログに記録
  - input: { category, source, direction, multiplier, priority }

updateRule(ruleId: string, input: Partial<RewardRuleInput>): RewardRule
  - ルール更新。監査ログに記録

deleteRule(ruleId: string): void
  - ルール削除。監査ログに記録

toggleRule(ruleId: string, enabled: boolean): RewardRule
  - ルール有効/無効切り替え。監査ログに記録
```

---

## AI-07: RewardRuleEngine【報酬勾配制御】

```
applyRules(category: string, source: DataSourceType, basePoints: number): AppliedRewardResult
  - 管理者定義ルールを優先度順に評価し、最初にマッチしたルールの倍率を適用する
  - マッチなしの場合は倍率 1.0 を使用
  - category が 'broad_impact' の場合は excludeFromSocialProof=true を付与する
  - 戻り値: { finalPoints: number, appliedRule: RewardRule | null, multiplier: number, excludeFromSocialProof: boolean }

getRules(): RewardRule[]
  - DynamoDB からルール一覧をキャッシュ付きで取得（TTL: 60秒）
```

---

## AI-08: CheckpointMaintenanceEngine【自律的枝刈り・PIIチェック】

```
scanPII(text: string): PIIScanResult
  - テキスト中の個人情報・固有名詞を検出する（リアルタイム・生成直後に呼び出し）
  - 検出対象: 氏名・メールアドレス・電話番号・固有プロジェクト名・会社名等
  - 戻り値: { hasPII: boolean, detectedItems: string[], canAnonymize: boolean }

anonymize(text: string): AnonymizeResult
  - 固有名詞・個人情報を一般名詞に置換する
  - 例: 「Aさんと会った」→「人と会った」
  - 置換不可能な場合は rejected=true を返す
  - 戻り値: { anonymized: string, rejected: boolean }

migrateAchievements(deletedId: string, survivorId: string | null): MigrationResult
  - deletedId を達成済みのユーザーを特定する
  - survivorId が存在する場合: そのユーザーの達成記録を survivorId に移行する
    （ポイント再計算なし・達成日時は元のものを保持）
  - survivorId が null の場合: デフォルトチェックポイント（「何かを達成しました」+10pt）の
    達成記録を追加する
  - デフォルトチェックポイントはメンテナンス対象外フラグ（isDefault=true）を持つ
  - 戻り値: { migratedToSurvivor: number, migratedToDefault: number }

runMaintenance(): MaintenanceResult
  - グローバルチェックポイントマスターをメンテナンスする
  - 削除前に必ず migrateAchievements() を呼び出して達成済みユーザーを移行する
  - isDefault=true のチェックポイントはすべての検出処理をスキップする
  - 戻り値: { deleted: number, archived: number, flagged: number, piiRemoved: number, migrated: number, summary: string }

detectDuplicates(): DuplicateGroup[]
  - LLMを使って類似チェックポイントをグループ化する
  - 戻り値: [{ survivorId: string, deleteIds: string[] }]

detectStale(thresholdDays: number): string[]
  - 指定日数以上達成実績ゼロのチェックポイントIDを返す（達成実績ありは対象外）

detectOverGeneration(windowMinutes: number, maxCount: number): OverGenerationResult[]
  - 指定時間窓内に大量生成されたチェックポイントの削除対象IDと存続IDを返す
  - 戻り値: [{ survivorId: string, deleteIds: string[] }]

flagLowQuality(checkpoints: Checkpoint[]): string[]
  - LLMで品質を評価し、不明瞭・不適切なチェックポイントのIDを返す
```
