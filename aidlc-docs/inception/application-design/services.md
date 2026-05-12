# サービス定義

## SVC-01: UserService
**責務**: ユーザー識別・認証の統合管理

**オーケストレーション**:
- Phase 1: メールアドレスによる簡易識別
- Phase 1.5: Cognito認証への移行（認証レイヤーを差し替え可能な構造）

**依存コンポーネント**: BE-01 UserHandler, INF-03 Database

---

## SVC-02: CheckpointService
**責務**: チェックポイントのライフサイクル管理

**オーケストレーション**:
1. AI-04 CheckpointGenerator からチェックポイント生成結果を受け取る
2. BE-02 CheckpointHandler 経由でDBに登録
3. 達成判定が完了したら BE-03 PointHandler にポイント加算を依頼
4. BE-08 SSEHandler 経由でフロントエンドに更新通知

**依存コンポーネント**: BE-02, BE-03, BE-08, AI-04, INF-02, INF-03

---

## SVC-03: ChatService
**責務**: AIチャットの統合処理（ユーザーとシステムの唯一の対話窓口）

**オーケストレーション**:
1. BE-04 ChatHandler がリクエスト受信
2. 処理時間を推定:
   - 短時間（即時応答可能）→ AI-05 ChatResponder を同期呼び出し → 結果を直接返す
   - 長時間（バッチ処理必要）→ 即時ACKメッセージを返す → INF-02 キューに投入 → バックグラウンド処理 → BE-08 SSEHandler で完了通知
3. 振り返り入力・予定入力の両方を処理
4. チェックポイント生成が必要な場合は SVC-02 CheckpointService に委譲

**依存コンポーネント**: BE-04, AI-05, AI-01, INF-02, BE-08

---

## SVC-04: DataSourceIntegrationService
**責務**: 外部データソース連携の統合管理

**オーケストレーション**:
1. DS-01 OAuthManager でユーザーの権限付与を管理
2. Webhook受信（DS-02/03/04）→ DS-05 DataSourceEventNormalizer で正規化
3. 正規化イベントを INF-02 キューに投入
4. AI-04 CheckpointGenerator がキューからイベントを取得してチェックポイント生成
5. SVC-02 CheckpointService に生成結果を渡す

**Webhook優先方針**: Webhookが使えるソース（GitHub・Slack）はWebhook優先。Google CalendarはPush通知。ポーリングは最終手段。

**依存コンポーネント**: DS-01〜05, AI-04, INF-02, SVC-02

---

## SVC-05: AchievementService
**責務**: チェックポイント達成判定の統合管理

**オーケストレーション**:
1. データソースイベントまたはAIチャット入力をトリガーに非同期起動
2. BE-06 AchievementJudgeHandler が判定ロジックを実行
3. 達成確定 → BE-03 PointHandler にポイント加算
4. BE-08 SSEHandler でフロントエンドに達成通知

**依存コンポーネント**: BE-06, BE-03, BE-08, INF-02

---

## SVC-06: AdminService
**責務**: 管理者操作の統合管理

**オーケストレーション**:
- チェックポイント削除: BE-07 AdminHandler → BE-02 CheckpointHandler → ポイント取り消し → 監査ログ
- ユーザー削除: BE-07 AdminHandler → BE-01 UserHandler → 関連データ削除 → 監査ログ

**依存コンポーネント**: BE-07, BE-01, BE-02, BE-03, INF-03

---

## SVC-07: RealtimeNotificationService（Phase 1.4）
**責務**: リアルタイム達成通知の配信管理

**オーケストレーション**:
- Phase 1: BE-08 SSEHandler でダッシュボード更新通知
- Phase 1.4: ブラウザ通知・プッシュ通知への拡張

**依存コンポーネント**: BE-08

---

## SVC-08: AddictionEngineService【依存性強化】
**責務**: 依存性強化メカニズムの統合管理（可変報酬・ストリーク・損失回避通知）

**オーケストレーション**:

**可変報酬フロー**:
1. AchievementService から達成確定イベントを受け取る
2. `calculateVariableReward(basePoints)` で可変ポイントを計算
3. BE-03 PointHandler にポイント加算を依頼
4. AI-06 PraiseMessageGenerator で称賛メッセージを生成
5. BE-08 SSEHandler で `{ type: 'point_update', points, isBonus, praiseMessage }` を配信

**ストリーク管理フロー**:
1. ポイント加算時に `updateStreak(userId)` を呼び出す
2. ストリーク状態を INF-03 DynamoDB に記録
3. ダッシュボード表示時に `getStreakStatus(userId)` を返す

**損失回避通知フロー（毎日0時）**:
1. `checkDailyStreak()` をスケジューラーで起動
2. 当日未獲得ユーザーを抽出
3. BE-08 SSEHandler で損失回避メッセージを配信
4. Phase 1.4: SVC-07 RealtimeNotificationService 経由でプッシュ通知

**依存コンポーネント**: BE-03, BE-08, AI-06, INF-02, INF-03, SVC-05
