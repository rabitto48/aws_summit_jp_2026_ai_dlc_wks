# ユニット定義

## 技術選定サマリー

| ユニット | 言語/フレームワーク | 理由 |
|---|---|---|
| Unit 1: フロントエンド | Vite + React (TypeScript) + Tailwind CSS + shadcn/ui | シンプルなSPA・SSE対応・高いデザイン性 |
| Unit 2: バックエンドAPI | Go (Golang) | Lambda Cold Start 高速・goroutineによる並行処理・SSE実装効率 |
| Unit 3: AI/LLM層 | Python | LangChain・boto3(Bedrock)・Ollamaクライアントが最も充実 |
| Unit 4: データソース連携 | Go (Golang) | Unit 2と統一。Webhook受信・OAuth処理の並行処理に適合 |
| Unit 5: インフラ | TypeScript (AWS CDK) | CDKのTypeScriptサポートが最も成熟 |

---

## 開発戦略

**着手順序**: インフラ基盤（最小限）→ 各ユニット並列開発（モック活用）→ 結合テスト
- 最初に最低限デプロイ可能なインフラを構築
- 各ユニットはモックを活用して並列開発
- ユニット間の結合状況を確認する統合テストを用意

**リポジトリ構成**: モノレポ（分離可能な構造）

```
aws_summit_jp_2026_ai_dlc_wks/
├── frontend/          # Unit 1: Vite + React
├── backend/           # Unit 2: Go API
├── ai/                # Unit 3: Python LLM
├── datasource/        # Unit 4: Go Connectors
├── infra/             # Unit 5: AWS CDK
└── aidlc-docs/        # ドキュメント
```

---

## Unit 1: フロントエンド

**言語**: TypeScript / Vite + React  
**責務**: ユーザー向けWebアプリ全体

**含むコンポーネント**:
- FE-01: DashboardPage（ポイント可視化・ストリーク・認知の歪み演出）
- FE-02: ChatPage（AIチャット — 唯一の対話窓口）
- FE-03: SettingsPage（データソース連携設定）
- FE-04: AdminPage（管理者専用）
- FE-05〜07: 共有コンポーネント（PointDisplay・CheckpointCard・DataSourceConnector）
- **FE-08: StreakDisplay（依存性強化 — ストリーク表示・損失回避メッセージ）**
- **FE-09: PraiseMessage（認知の歪み演出 — 大げさな称賛メッセージ）**

**主要技術**:
- Vite + React (TypeScript)
- Tailwind CSS + shadcn/ui（コンポーネントライブラリ）
- REST API クライアント
- Server-Sent Events 受信（ブラウザ標準 EventSource API）

**ディレクトリ構成**:
```
frontend/
├── src/
│   ├── pages/
│   │   ├── Dashboard/
│   │   ├── Chat/
│   │   ├── Settings/
│   │   └── Admin/
│   ├── components/
│   │   └── shared/
│   └── lib/
│       └── api/
└── public/
```

---

## Unit 2: バックエンドAPI

**言語**: Go (Golang)  
**責務**: REST API エンドポイント群・SSE配信・ビジネスロジック

**含むコンポーネント**:
- BE-01: UserHandler
- BE-02: CheckpointHandler
- BE-03: PointHandler（可変報酬計算含む）
- BE-04: ChatHandler（同期/非同期振り分け）
- BE-06: AchievementJudgeHandler
- BE-07: AdminHandler
- BE-08: SSEHandler
- **BE-09: RewardRuleHandler（報酬勾配ルールCRUD）**
- **SVC-08: AddictionEngineService（依存性強化 — 可変報酬・ストリーク管理・損失回避通知）**

**主要技術**:
- Go + AWS Lambda (provided.al2023)
- API Gateway（REST + SSE）
- AWS SDK for Go v2
- SQS（非同期キュー投入）
- DynamoDB クライアント

**ディレクトリ構成**:
```
backend/
├── cmd/
│   └── lambda/
├── internal/
│   ├── handler/
│   ├── service/
│   ├── repository/
│   └── middleware/
└── pkg/
    └── model/
```

---

## Unit 3: AI/LLM層

**言語**: Python  
**責務**: チェックポイント生成・AIチャット応答・LLMバックエンド抽象化

**含むコンポーネント**:
- AI-01: LLMProvider（インターフェース + Strategy Pattern）
- AI-02: LocalLLMAdapter（Ollama）
- AI-03: BedrockAdapter（Amazon Bedrock）
- AI-04: CheckpointGenerator（非同期）
- AI-05: ChatResponder（同期/非同期）
- **AI-06: PraiseMessageGenerator（認知の歪み演出 — 称賛メッセージ生成）**
- **AI-07: RewardRuleEngine（報酬勾配ルール適用エンジン）**
- **AI-08: CheckpointMaintenanceEngine（自律的枝刈り・メンテナンス）**

**主要技術**:
- Python 3.12 + AWS Lambda
- boto3（Amazon Bedrock）
- Ollama Python クライアント
- LangChain（オプション）
- SQS（キュー受信）
- 環境変数: `LLM_BACKEND=local|bedrock`

**ディレクトリ構成**:
```
ai/
├── src/
│   ├── provider/
│   │   ├── base.py
│   │   ├── local_llm.py
│   │   └── bedrock.py
│   ├── generator/
│   └── responder/
└── tests/
```

---

## Unit 4: データソース連携

**言語**: Go (Golang)  
**責務**: 外部データソースとの連携・Webhook受信・OAuth管理・イベント正規化

**Phase 1 スコープ**: Slack のみ  
**Wave 1 スコープ**: GitHub・Google Calendar・Google Meet・Discord を追加  
**Wave 2以降**: 生産性ツール・ライフスタイル・IoT（Wave 2〜4）

**含むコンポーネント**:
- DS-01: OAuthManager
- DS-02: GitHubConnector（Phase 1.7）
- DS-03: GoogleCalendarConnector（Google Meet検出含む）（Phase 1.7）
- DS-04: SlackConnector（Phase 1）
- DS-05: DataSourceEventNormalizer
- BE-05: DataSourceWebhookHandler

**主要技術**:
- Go + AWS Lambda (provided.al2023)
- API Gateway（Webhookエンドポイント）
- SQS（正規化イベント投入）
- DynamoDB（OAuthトークン保存）
- Webhook署名検証（GitHub HMAC・Slack signing secret）

**ディレクトリ構成**:
```
datasource/
├── cmd/
│   └── lambda/
├── internal/
│   ├── connector/
│   │   ├── github/
│   │   ├── gcalendar/
│   │   └── slack/
│   ├── oauth/
│   └── normalizer/
└── pkg/
    └── model/
```

---

## Unit 5: インフラ

**言語**: TypeScript (AWS CDK)  
**責務**: AWSリソース定義・Docker開発環境・CI/CD基盤

**含むコンポーネント**:
- INF-01: APIGateway（REST + SSEエンドポイント）
- INF-02: SQS（非同期キュー）
- INF-03: DynamoDB（テーブル定義）
- INF-04: Docker Compose（Ollama + LocalStack + 各サービス）

**DynamoDBテーブル定義**:

| テーブル名 | PK | SK | 主な属性 | 用途 |
|---|---|---|---|---|
| `checkpoints` | `checkpointId` | - | `category`, `basePoints`, `isDefault`, `excludeFromSocialProof`, `createdAt` | グローバルチェックポイントマスター |
| `user_achievements` | `userId` | `checkpointId#achievedAt` | `points`, `isBonus`, `praiseMessage` | ユーザーごとの達成記録 |
| `point_history` | `userId` | `timestamp` | `delta`, `total`, `checkpointId`, `category` | ポイント加減算履歴（マイナス含む） |
| `streaks` | `userId` | - | `currentStreak`, `longestStreak`, `lastAchievedDate` | ストリーク管理 |
| `broad_impact_log` | `userId` | `timestamp` | `checkpointId`, `penaltyMultiplier` | broad_impact行動の累積ペナルティ管理 |
| `reward_rules` | `ruleId` | - | `category`, `source`, `multiplier`, `priority`, `enabled` | 管理者定義の報酬勾配ルール |
| `maintenance_summary` | `runId` | - | `deleted`, `archived`, `piiRemoved`, `migrated`, `runAt` | メンテナンス実行サマリー |
| `oauth_tokens` | `userId` | `source` | `accessToken`, `refreshToken`, `expiresAt` | OAuthトークン（暗号化保存） |

**主要技術**:
- AWS CDK v2 (TypeScript)
- Docker Compose（ローカル開発）
- LocalStack（AWSサービスエミュレーション）
- Ollama（ローカルLLM）

**ディレクトリ構成**:
```
infra/
├── lib/
│   ├── api-stack.ts
│   ├── database-stack.ts
│   ├── queue-stack.ts
│   └── datasource-stack.ts
├── bin/
└── docker/
    ├── docker-compose.yml
    └── docker-compose.dev.yml
```
