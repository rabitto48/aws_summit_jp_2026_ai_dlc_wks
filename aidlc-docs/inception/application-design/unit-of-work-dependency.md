# ユニット依存関係

## 依存関係マトリクス

| ユニット | 依存先 | 依存種別 |
|---|---|---|
| Unit 1: フロントエンド | Unit 2: バックエンドAPI | REST API + SSE（ランタイム） |
| Unit 2: バックエンドAPI | Unit 3: AI/LLM層 | SQSキュー経由（非同期） |
| Unit 2: バックエンドAPI | Unit 5: インフラ | DynamoDB・SQS（ランタイム） |
| Unit 3: AI/LLM層 | Unit 5: インフラ | SQS・DynamoDB（ランタイム） |
| Unit 4: データソース連携 | Unit 2: バックエンドAPI | SQSキュー経由（非同期） |
| Unit 4: データソース連携 | Unit 5: インフラ | DynamoDB・SQS（ランタイム） |
| Unit 5: インフラ | なし | 最上流（依存なし） |

## 開発順序（依存関係に基づく）

```
Phase A（先行）:
  Unit 5: インフラ（最小限）
    └─ DynamoDB テーブル・SQSキュー・API Gateway の基盤のみ

Phase B（並列開発）:
  Unit 2: バックエンドAPI  ─┐
  Unit 3: AI/LLM層        ─┤ モックを活用して並列開発
  Unit 4: データソース連携  ─┘

Phase C（フロントエンド）:
  Unit 1: フロントエンド
    └─ Unit 2 の API 仕様が固まり次第開始（モックAPIで先行可）

Phase D（インフラ追加）:
  Unit 5: インフラ（追加）
    └─ 各ユニットの要件に応じて随時追加

Phase E（結合テスト）:
  全ユニット統合テスト
```

## ユニット間インターフェース

### Unit 1 ↔ Unit 2
- **プロトコル**: REST API (HTTPS) + Server-Sent Events
- **認証**: メールアドレスヘッダー（Phase 1）
- **API仕様**: OpenAPI 3.0 で定義（Unit 2が提供）

### Unit 2 ↔ Unit 3
- **プロトコル**: SQS メッセージ（JSON）
- **メッセージ形式**: `{ type: "generate_checkpoint" | "chat_response", payload: NormalizedEvent | ChatRequest }`
- **応答**: SQS → Lambda → SSE（Unit 2経由でフロントエンドへ）

### Unit 2 ↔ Unit 4
- **プロトコル**: SQS メッセージ（JSON）
- **メッセージ形式**: `{ source: DataSourceType, userId: string, event: NormalizedEvent }`

### Unit 2/3/4 ↔ Unit 5
- **DynamoDB**: テーブル名・GSI定義を Unit 5 が提供
- **SQS**: キューURL を環境変数で注入
- **共有モデル**: `pkg/model/` に型定義を配置（Go ユニット間）

## モック戦略

| ユニット | 開発時モック |
|---|---|
| Unit 1 | MSW（Mock Service Worker）でUnit 2 APIをモック |
| Unit 2 | LocalStack で DynamoDB・SQS をエミュレート |
| Unit 3 | Ollama（ローカルLLM）で Bedrock をモック |
| Unit 4 | ngrok + テスト用Webhook送信でローカル受信テスト |
