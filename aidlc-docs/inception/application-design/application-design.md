# アプリケーション設計（統合ドキュメント）

**作成日**: 2026-05-09（再設計: 2026-05-09）  
**対象**: Phase 1（Webアプリ本体）  
**テーマ**: 「人をダメにするサービス」— 依存性設計・可変報酬・認知の歪み演出を設計に組み込む

---

## 1. 設計方針サマリー

| 項目 | 決定内容 |
|---|---|
| フロントエンド構成 | ページ構成 + 再利用可能な機能コンポーネントのハイブリッド |
| バックエンドAPI構成 | 処理ごとに差し替え可能なアダプター構造（疎結合） |
| AI/LLM処理 | チャット: 短時間=同期・長時間=即時ACK+非同期。チェックポイント生成・達成判定: 常に非同期 |
| データソース連携 | Webhook/Push通知優先。ポーリングは最終手段 |
| フロント↔バック通信 | REST API + Server-Sent Events（リアルタイム更新） |
| 管理者機能 | 同一バックエンドAPIを共有、フロントエンドのみ分離 |
| LLMバックエンド | Strategy Pattern + 環境変数（LLM_BACKEND=local\|bedrock）で切り替え |
| **可変報酬設計** | ポイント付与量をランダム変動（±0〜50%）させ、期待感を生む |
| **依存性設計** | ストリーク管理・損失回避メッセージ・毎日開く動機を設計に組み込む |
| **認知の歪み演出** | 些細な行動と重要な成果を同一UIで並列表示し、差を小さく見せる |

---

## 2. コンポーネント構成

### 2.1 フロントエンド（4ページ + 5共有コンポーネント）

```
フロントエンド
├── Pages
│   ├── FE-01: DashboardPage      # ポイント可視化・ストリーク・達成一覧・認知の歪み演出
│   ├── FE-02: ChatPage           # AIチャット（唯一の対話窓口）
│   ├── FE-03: SettingsPage       # データソース連携設定
│   └── FE-04: AdminPage          # 管理者専用（フロントエンドのみ分離）
└── Shared Components
    ├── FE-05: PointDisplay        # ポイント数値表示（カウントアップアニメーション付き）
    ├── FE-06: CheckpointCard      # 達成チェックポイントカード（些細/重要を同一UIで表示）
    ├── FE-07: DataSourceConnector # OAuth連携ボタン・状態表示
    ├── FE-08: StreakDisplay        # ストリーク表示・損失回避メッセージ（新規）
    └── FE-09: PraiseMessage       # 大げさな称賛メッセージ表示（新規）
```

### 2.2 バックエンド（8ハンドラー）

```
バックエンド（API Gateway + Lambda）
├── BE-01: UserHandler             # ユーザー識別・認証（差し替え可能）
├── BE-02: CheckpointHandler       # チェックポイントCRUD（システム/管理者のみ）
├── BE-03: PointHandler            # ポイント集計・履歴（可変報酬計算含む）
├── BE-04: ChatHandler             # AIチャット受付・同期/非同期振り分け
├── BE-05: DataSourceWebhookHandler # Webhook受信・署名検証
├── BE-06: AchievementJudgeHandler  # 達成判定
├── BE-07: AdminHandler            # 管理者操作・監査ログ
├── BE-08: SSEHandler              # Server-Sent Events配信
└── BE-09: RewardRuleHandler       # 報酬勾配ルールCRUD（管理者のみ）（新規）
```

### 2.3 AI/LLM層（Strategy Pattern）

```
AI/LLM層
├── AI-01: LLMProvider（インターフェース）
│   ├── AI-02: LocalLLMAdapter    # Ollama（開発環境・プライバシー配慮）
│   └── AI-03: BedrockAdapter     # Amazon Bedrock（本番・高精度）
├── AI-04: CheckpointGenerator    # チェックポイント生成（非同期・細分化優先）
├── AI-05: ChatResponder          # チャット応答生成（同期/非同期）
├── AI-06: PraiseMessageGenerator # 演出メッセージ生成（4段階対応）
├── AI-07: RewardRuleEngine       # 報酬勾配ルール適用エンジン
└── AI-08: CheckpointMaintenanceEngine # チェックポイント枝刈り・メンテナンス（新規）
```

### 2.4 データソース連携層（プラグイン拡張可能）

```
データソース連携層
├── DS-01: OAuthManager           # OAuth 2.0認可フロー管理
├── DS-02: GitHubConnector        # GitHub Webhook
├── DS-03: GoogleCalendarConnector # Google Calendar Push通知 + Meet検出
├── DS-04: SlackConnector         # Slack Events API
└── DS-05: DataSourceEventNormalizer # 統一フォーマット変換
```

### 2.5 インフラ

```
インフラ
├── INF-01: APIGateway            # REST + SSEエンドポイント
├── INF-02: MessageQueue（SQS）   # 非同期処理キュー
├── INF-03: Database（DynamoDB）  # 永続化（ストリーク・可変報酬履歴含む）
└── INF-04: DockerCompose         # ローカル開発環境（Ollama含む）
```

---

## 3. 新規コンポーネント詳細

### FE-08: StreakDisplay（依存性強化コンポーネント）
**責務**: ストリーク（連続記録）の表示と損失回避メッセージの演出

**表示内容**:
- 「X日連続ポイント獲得中🔥」を大きく表示
- 当日未獲得の場合: 「今日まだポイントを獲得していません。ストリークが途切れます」
- ストリーク途切れ後: 「記録が途切れました。今日から再スタート！」（再依存を促す）
- ストリーク最長記録の表示

**設計意図**: 損失回避（Loss Aversion）を利用してアプリを毎日開く動機を作る

### FE-09: PraiseMessage（認知の歪み演出コンポーネント）
**責務**: チェックポイント達成時・ダッシュボード表示時の大げさな称賛メッセージ表示

**表示内容**:
- チェックポイント達成時: 行動種別に応じた称賛メッセージ（AI生成・毎回微妙に異なる）
- ダッシュボード表示時: 「今日も頑張りました！」「完璧な一日！」等
- ボーナスポイント時: 「ラッキーボーナス！+Xpt」の演出

**設計意図**: 些細な行動を偉業として演出し、認知の歪みを強化する

### AI-06: PraiseMessageGenerator（称賛メッセージ生成）
**責務**: チェックポイントの種類・ポイント量・ユーザーコンテキストに応じた大げさな称賛メッセージをLLMで生成

**入力**: チェックポイント種別、獲得ポイント、ユーザーの最近の行動履歴  
**出力**: 称賛メッセージ文字列（毎回微妙に異なる表現）  
**LLM使用**: LocalLLMAdapter（低レイテンシ優先）

---

## 4. サービス層（オーケストレーション）

| サービス | 責務 |
|---|---|
| SVC-01: UserService | ユーザー識別・認証統合 |
| SVC-02: CheckpointService | チェックポイントライフサイクル管理 |
| SVC-03: ChatService | AIチャット統合処理（同期/非同期振り分け） |
| SVC-04: DataSourceIntegrationService | データソース連携統合 |
| SVC-05: AchievementService | 達成判定統合・可変報酬計算 |
| SVC-06: AdminService | 管理者操作統合 |
| SVC-07: RealtimeNotificationService | リアルタイム通知（Phase 1.4で拡張） |
| **SVC-08: AddictionEngineService** | **依存性強化統合（ストリーク管理・損失回避通知・可変報酬計算・broad_impact累積ペナルティ）（Unit 2実装。broad_impact判定はUnit 3 AI-07と連携）** |

---

## 5. 主要データフロー

### データソース → チェックポイント生成 → ポイント付与（停滞報酬・可変報酬）
```
外部ソース → Webhook受信 → イベント正規化 → SQSキュー
→ CheckpointGenerator（LLM）:
    1. 行動を分類（stagnation/retreat/personal_completion/broad_impact）
    2. [broad_impact の場合] 即時SSEで事前警告をダッシュボードに配信
    3. 固有名詞・個人情報を匿名化（不可能なら生成中止）
    4. PIIスキャン（検出されたら生成中止）
    5. グローバルマスターに類似チェックポイントがあれば再利用
→ RewardRuleEngine: 管理者定義ルールを参照 → ポイント倍率を決定
→ [broad_impact の場合] AddictionEngineService.applyBroadImpactPenalty(): 累積ペナルティ倍率を適用
→ チェックポイント登録（グローバルマスター）
→ AchievementJudge → AddictionEngineService: stagnation/retreatのみ可変報酬を追加適用
→ PraiseMessageGenerator（LLM） → 称賛メッセージ生成
→ SSE → ダッシュボード更新
```

### AIチャット（振り返り・予定入力）
```
ChatPage → ChatHandler
  ├─[短時間] → ChatResponder（同期） → 即時応答
  └─[長時間] → 即時ACK → SQSキュー → ChatResponder（非同期）
               → SSE → ChatPage（完了通知）
```

### 依存性強化フロー（AddictionEngineService）
```
毎日0時: ストリーク状態チェック
  → 当日未獲得ユーザー: 損失回避メッセージをSSEで配信（Phase 1.4: プッシュ通知）
  → ストリーク更新: DynamoDB に連続日数を記録
  → ダッシュボード表示時: StreakDisplay コンポーネントに最新状態を返す
```

### チェックポイントメンテナンスフロー（CheckpointMaintenanceEngine）
```
定期バッチ（毎日深夜）または閾値超過トリガー:
  → CheckpointMaintenanceEngine 起動
  → 重複検出: グローバルマスター内の類似チェックポイントを LLM で判定 → 低ポイント側を削除
  → 陳腐化検出: 30日以上達成実績ゼロのチェックポイントをアーカイブ
  → 過剰生成検出: 短時間大量生成を検出 → 上位N件のみ残して削除
  → 低品質検出: 不明瞭・不適切なチェックポイントを管理者レビューキューに追加
  → メンテナンスサマリーを DynamoDB に記録 → 管理者画面に表示
```

### 可変報酬計算フロー
```
AchievementJudge → 達成確定
→ AddictionEngineService.calculateVariableReward(basePoints)
  → basePoints × (1.0 + random(0, 0.5)) を計算
  → ボーナス判定（20%の確率でボーナスポイント演出）
→ PointHandler.addPoints(userId, checkpointId, calculatedPoints)
→ SSE: { type: 'point_update', points, isBonus, praiseMessage }
```

---

## 6. 詳細ドキュメント参照

- コンポーネント詳細: `components.md`
- メソッドシグネチャ: `component-methods.md`
- サービス詳細: `services.md`
- 依存関係・データフロー: `component-dependency.md`
