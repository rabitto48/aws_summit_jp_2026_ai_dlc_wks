# アプリケーション設計（統合ドキュメント）

**作成日**: 2026-05-09  
**対象**: Phase 1（Webアプリ本体）

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

---

## 2. コンポーネント構成

### 2.1 フロントエンド（4ページ + 3共有コンポーネント）

```
フロントエンド
├── Pages
│   ├── FE-01: DashboardPage      # ポイント可視化・達成一覧・グラフ
│   ├── FE-02: ChatPage           # AIチャット（唯一の対話窓口）
│   ├── FE-03: SettingsPage       # データソース連携設定
│   └── FE-04: AdminPage          # 管理者専用（フロントエンドのみ分離）
└── Shared Components
    ├── FE-05: PointDisplay        # ポイント数値表示
    ├── FE-06: CheckpointCard      # 達成チェックポイントカード
    └── FE-07: DataSourceConnector # OAuth連携ボタン・状態表示
```

### 2.2 バックエンド（8ハンドラー）

```
バックエンド（API Gateway + Lambda）
├── BE-01: UserHandler             # ユーザー識別・認証（差し替え可能）
├── BE-02: CheckpointHandler       # チェックポイントCRUD（システム/管理者のみ）
├── BE-03: PointHandler            # ポイント集計・履歴
├── BE-04: ChatHandler             # AIチャット受付・同期/非同期振り分け
├── BE-05: DataSourceWebhookHandler # Webhook受信・署名検証
├── BE-06: AchievementJudgeHandler  # 達成判定
├── BE-07: AdminHandler            # 管理者操作・監査ログ
└── BE-08: SSEHandler              # Server-Sent Events配信
```

### 2.3 AI/LLM層（Strategy Pattern）

```
AI/LLM層
├── AI-01: LLMProvider（インターフェース）
│   ├── AI-02: LocalLLMAdapter    # Ollama（開発環境・プライバシー配慮）
│   └── AI-03: BedrockAdapter     # Amazon Bedrock（本番・高精度）
├── AI-04: CheckpointGenerator    # チェックポイント生成（非同期）
└── AI-05: ChatResponder          # チャット応答生成（同期/非同期）
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
├── INF-03: Database（DynamoDB）  # 永続化
└── INF-04: DockerCompose         # ローカル開発環境（Ollama含む）
```

---

## 3. サービス層（オーケストレーション）

| サービス | 責務 |
|---|---|
| SVC-01: UserService | ユーザー識別・認証統合 |
| SVC-02: CheckpointService | チェックポイントライフサイクル管理 |
| SVC-03: ChatService | AIチャット統合処理（同期/非同期振り分け） |
| SVC-04: DataSourceIntegrationService | データソース連携統合 |
| SVC-05: AchievementService | 達成判定統合 |
| SVC-06: AdminService | 管理者操作統合 |
| SVC-07: RealtimeNotificationService | リアルタイム通知（Phase 1.4で拡張） |

---

## 4. 主要データフロー

### データソース → チェックポイント生成 → ポイント付与
```
外部ソース → Webhook受信 → イベント正規化 → SQSキュー
→ CheckpointGenerator（LLM） → チェックポイント登録
→ AchievementJudge → ポイント加算 → SSE → ダッシュボード更新
```

### AIチャット（振り返り・予定入力）
```
ChatPage → ChatHandler
  ├─[短時間] → ChatResponder（同期） → 即時応答
  └─[長時間] → 即時ACK → SQSキュー → ChatResponder（非同期）
               → SSE → ChatPage（完了通知）
```

---

## 5. 詳細ドキュメント参照

- コンポーネント詳細: `components.md`
- メソッドシグネチャ: `component-methods.md`
- サービス詳細: `services.md`
- 依存関係・データフロー: `component-dependency.md`
