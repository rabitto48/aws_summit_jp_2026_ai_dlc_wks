# ユニット分解計画

## 実行チェックリスト

### PART 1: Planning
- [x] Step 1: コンテキスト分析（Application Design成果物読み込み）
- [x] Step 2: ユニット分解計画作成
- [x] Step 3: 質問生成
- [x] Step 4: ユーザー回答収集
- [x] Step 5: 回答分析・曖昧さ解消（技術選定承認済み）
- [x] Step 6: 計画承認

### PART 2: Generation
- [x] Step 7: unit-of-work.md 生成
- [x] Step 8: unit-of-work-dependency.md 生成
- [x] Step 9: unit-of-work-story-map.md 生成
- [ ] Step 10: 完了確認・承認待ち

---

## 想定ユニット構成（Application Designより）

Application Design で特定した4層・18コンポーネントを以下の5ユニット候補に分解する案：

| ユニット候補 | 含むコンポーネント |
|---|---|
| Unit 1: フロントエンド | FE-01〜07（Webアプリ全体） |
| Unit 2: バックエンドAPI | BE-01〜08（API Gateway + Lambda群） |
| Unit 3: AI/LLM層 | AI-01〜05（LLMProvider・Generator・Responder） |
| Unit 4: データソース連携 | DS-01〜05（OAuth・各Connector・Normalizer） |
| Unit 5: インフラ | INF-01〜04（AWS CDK/SAM・Docker・SQS・DynamoDB） |

---

