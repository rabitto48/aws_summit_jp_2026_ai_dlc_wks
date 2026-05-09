# ユニット × ストーリーマップ

## マッピング一覧

| ストーリー | タイトル | 主担当ユニット | 関連ユニット |
|---|---|---|---|
| US-01 | メールアドレスによるユーザー識別 | Unit 2 (API) | Unit 1 (FE), Unit 5 (Infra) |
| US-02 | データソース連携の権限付与 | Unit 4 (DS) | Unit 1 (FE), Unit 2 (API), Unit 5 (Infra) |
| US-03 | AIとの対話で行動後の振り返りを入力 | Unit 3 (AI) | Unit 1 (FE), Unit 2 (API) |
| US-04 | AIとの対話で行動予定を入力 | Unit 3 (AI) | Unit 1 (FE), Unit 2 (API) |
| US-06 | Google Calendarからチェックポイント自動生成 | Unit 4 (DS) | Unit 3 (AI), Unit 2 (API) |
| US-07 | GitHubからチェックポイント自動生成 | Unit 4 (DS) | Unit 3 (AI), Unit 2 (API) |
| US-08 | Slackからチェックポイント自動生成 | Unit 4 (DS) | Unit 3 (AI), Unit 2 (API) |
| US-09 | Google Meetからチェックポイント自動生成 | Unit 4 (DS) | Unit 3 (AI), Unit 2 (API) |
| US-10 | システムが自律的に達成判定を行う | Unit 2 (API) | Unit 3 (AI), Unit 5 (Infra) |
| US-11 | 達成済みチェックポイントのみ閲覧できる | Unit 2 (API) | Unit 1 (FE) |
| US-12 | 累計ポイントを大きく確認できる | Unit 1 (FE) | Unit 2 (API) |
| US-13 | 最近達成したチェックポイントを確認できる | Unit 1 (FE) | Unit 2 (API) |
| US-14 | ポイント推移グラフを確認できる | Unit 1 (FE) | Unit 2 (API) |
| US-15 | ポイント蓄積でレベルアップする | Unit 2 (API) | Unit 1 (FE) |
| US-16 | 達成バッジを獲得できる | Unit 2 (API) | Unit 1 (FE) |
| US-17 | AIが生成したチェックポイントをレビューできる | Unit 2 (API) | Unit 1 (FE/Admin) |
| US-18 | 不適切なチェックポイントを削除できる | Unit 2 (API) | Unit 1 (FE/Admin) |
| US-19 | 緊急時にユーザーを削除できる | Unit 2 (API) | Unit 1 (FE/Admin) |
| US-20 | 管理者としてシステムにアクセスできる | Unit 2 (API) | Unit 1 (FE/Admin), Unit 5 (Infra) |

---

## ユニット別ストーリー集計

### Unit 1: フロントエンド（主担当）
US-12, US-13, US-14

### Unit 2: バックエンドAPI（主担当）
US-01, US-10, US-11, US-15, US-16, US-17, US-18, US-19, US-20

### Unit 3: AI/LLM層（主担当）
US-03, US-04

### Unit 4: データソース連携（主担当）
US-02, US-06, US-07, US-08, US-09

### Unit 5: インフラ（主担当）
なし（全ユニットの基盤として横断的に関与）

---

## 開発フェーズ別ストーリー

### Phase A: インフラ基盤
- インフラ基盤構築（ストーリー対応なし、全ユニットの前提）

### Phase B: 並列開発
**Unit 2**: US-01, US-10, US-11, US-15, US-16, US-17, US-18, US-19, US-20  
**Unit 3**: US-03, US-04  
**Unit 4**: US-02, US-06, US-07, US-08, US-09  

### Phase C: フロントエンド
**Unit 1**: US-12, US-13, US-14（Unit 2 API確定後）

### Phase E: 結合テスト
全ストーリーの End-to-End 検証
