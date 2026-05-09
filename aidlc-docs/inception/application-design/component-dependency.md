# コンポーネント依存関係

## 依存関係マトリクス

| コンポーネント | 依存先 | 通信パターン |
|---|---|---|
| FE-01 DashboardPage | BE-03 PointHandler, BE-02 CheckpointHandler, BE-08 SSEHandler | REST + SSE |
| FE-02 ChatPage | BE-04 ChatHandler, BE-08 SSEHandler | REST + SSE |
| FE-03 SettingsPage | DS-01 OAuthManager, BE-01 UserHandler | REST |
| FE-04 AdminPage | BE-07 AdminHandler | REST |
| BE-04 ChatHandler | AI-05 ChatResponder, INF-02 Queue, BE-08 SSEHandler | 同期/非同期 |
| BE-05 WebhookHandler | DS-02/03/04 Connectors | 同期 |
| BE-06 AchievementJudgeHandler | BE-03 PointHandler, BE-08 SSEHandler | 同期 |
| BE-07 AdminHandler | BE-02 CheckpointHandler, BE-01 UserHandler, BE-03 PointHandler | 同期 |
| AI-04 CheckpointGenerator | AI-01 LLMProvider, INF-02 Queue | 非同期 |
| AI-05 ChatResponder | AI-01 LLMProvider | 同期/非同期 |
| AI-01 LLMProvider | AI-02 LocalLLMAdapter または AI-03 BedrockAdapter | Strategy Pattern |
| DS-02/03/04 Connectors | DS-05 DataSourceEventNormalizer | 同期 |
| DS-05 Normalizer | INF-02 Queue | 非同期 |
| SVC-02 CheckpointService | BE-02, BE-03, BE-08, AI-04, INF-02, INF-03 | 混在 |
| SVC-03 ChatService | BE-04, AI-05, INF-02, BE-08 | 混在 |
| SVC-04 DataSourceIntegrationService | DS-01〜05, AI-04, INF-02, SVC-02 | 混在 |

---

## データフロー図

### フロー1: データソース連携 → チェックポイント生成 → ポイント付与

```
外部ソース（GitHub/Slack/Calendar）
  │ Webhook / Push通知
  ▼
BE-05 DataSourceWebhookHandler
  │ 署名検証
  ▼
DS-02/03/04 Connectors
  │ イベント受信
  ▼
DS-05 DataSourceEventNormalizer
  │ 統一フォーマットに変換
  ▼
INF-02 MessageQueue（SQS）
  │ 非同期キュー
  ▼
AI-04 CheckpointGenerator
  │ LLMProvider経由でチェックポイント生成
  ▼
SVC-02 CheckpointService
  │ チェックポイント登録
  ▼
BE-06 AchievementJudgeHandler
  │ 達成判定
  ▼
BE-03 PointHandler ──→ INF-03 Database（ポイント保存）
  │
  ▼
BE-08 SSEHandler ──→ FE-01 DashboardPage（リアルタイム更新）
```

### フロー2: AIチャット（振り返り・予定入力）

```
FE-02 ChatPage
  │ REST POST /chat
  ▼
BE-04 ChatHandler
  │
  ├─[短時間処理]──→ AI-05 ChatResponder ──→ AI-01 LLMProvider
  │                    │ 同期応答
  │                    ▼
  │                 FE-02 ChatPage（即時表示）
  │
  └─[長時間処理]──→ 即時ACK返却 ──→ FE-02 ChatPage（「処理中...」表示）
                    │
                    ▼
                 INF-02 Queue
                    │ 非同期
                    ▼
                 AI-05 ChatResponder ──→ AI-01 LLMProvider
                    │ 完了
                    ▼
                 BE-08 SSEHandler ──→ FE-02 ChatPage（完了通知）
                    │
                    ▼（チェックポイント生成が必要な場合）
                 SVC-02 CheckpointService → BE-03 PointHandler
```

### フロー3: LLMバックエンド切り替え（Strategy Pattern）

```
AI-04 CheckpointGenerator / AI-05 ChatResponder
  │ AI-01 LLMProvider インターフェース経由
  ▼
LLM_BACKEND 環境変数
  ├─[local] ──→ AI-02 LocalLLMAdapter ──→ Ollama（Docker）
  └─[bedrock] ──→ AI-03 BedrockAdapter ──→ Amazon Bedrock
```

---

## 疎結合設計の方針

- **LLM抽象レイヤー**: AI-01 LLMProvider インターフェースにより、ローカルLLM↔Bedrockを設定で切り替え可能
- **データソース抽象レイヤー**: DS-05 DataSourceEventNormalizer により、新ソース追加時は Connector + Normalizer のみ追加
- **認証レイヤー**: BE-01 UserHandler を差し替え可能な構造（Phase 1: メールアドレス → Phase 1.5: Cognito）
- **非同期分離**: INF-02 Queue により、LLM処理・チェックポイント生成・達成判定をバックエンドから分離
