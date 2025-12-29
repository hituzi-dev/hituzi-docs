# dating-app: システム構成

このドキュメントは企画書（`企画書/dating-app.md`）に基づき、実装・運用を想定したシステム構成を詳細にまとめたものです。MVP を想定しつつコスト最適化・テスト・デプロイを考慮しています。

---

## 全体アーキテクチャ（高レベル）

- モバイル（SP）: React Native
- Web FE: Vue
- Backend: Java（AWS Lambda）
- データストア: DynamoDB（必要に応じて Aurora Serverless を検討）
- 認証: Amazon Cognito（社内SSO：SAML/IdP連携を想定）
- API: API Gateway + Lambda
- 静的ホスティング: S3 + CloudFront
- CI/CD: GitHub Actions
- IaC: Terraform（主要リソースは Terraform で管理）
- ローカル統合テスト: LocalStack（AWS 模擬）
- 監視: CloudWatch（ログ / メトリクス）、必要に応じて Opensearch/Datadog 等
- 通知: SNS / Amazon Pinpoint（リマインダー・プッシュ通知）

---

## コンポーネント詳細

### 認証／ユーザ管理

- フロー: SP/FE -> Cognito (OAuth2 / OpenID Connect) -> Lambda (必要なら認可チェック)
- 社内向け: SAML / IdP を Cognito に連携して SSO を実現
- ユーザ属性: 社員番号、部署、表示名等を Cognito のカスタム属性に同期

### API 層

- 公開 API: API Gateway (REST) を前段に配置
- Lambda: Java ベースでビジネスロジックを実装（クリーンアーキテクチャ採用）
- 課金対策: Lambda のメモリ・タイムアウトを最小化しコールドスタート抑制（必要なら Provisioned Concurrency）

### データ層

- 主: DynamoDB（オンデマンド）
  - Users テーブル
  - PairRecord テーブル
  - 必要に応じて GSI を設計（週集計や未実施者クエリ用）
- RDB が必要な場合の代替: Aurora Serverless（コストとクエリ要件を比較）

### 通知／スケジューラ

- リマインダーは EventBridge（Cron） -> Lambda -> SNS/Pinpoint -> エンドポイント（メール/プッシュ）
- プッシュ通知: FCM/APNS を Pinpoint または独自 Lambda で送信

### 監視・ログ

- Lambda ログ: CloudWatch Logs
- メトリクス: CloudWatch Alarms（エラー率・レイテンシ閾値）
- 監査ログ: 必要に応じて S3 に保存

### セキュリティ

- IAM 権限最小化（Lambda ロールは必要最小限の DynamoDB 権限のみ）
- API Gateway で WAF を検討（社内限定なら不要）
- Secrets 管理: AWS Secrets Manager または Parameter Store

---

## テスト戦略（具体）

- 単体テスト（Unit）:
  - BE: JUnit + Mockito（ビジネスロジック／ユースケースに注力）
  - FE (Vue): Vitest + Vue Testing Library
  - SP (RN): Jest + React Native Testing Library
- 統合テスト:
  - LocalStack を利用した Lambda + DynamoDB の統合テスト（GitHub Actions 上で実行）
- E2E:
  - Web: Vitest / Playwright（UI 操作の E2E を想定）
  - Mobile: Maestro（企画書指定）を用いた端末上でのフロー検証
- カバレッジ閾値: 重要ビジネスロジックで高め（例: 80%）を目安に設定

---

## デプロイ＆CI/CD

- リポジトリごとの GitHub Actions:
  - `dating-app-be`: ビルド（Maven/Gradle）→ Unit テスト → Static analysis → deploy (Terraform apply / SAM deploy for infra changes)
  - `dating-app-fe`: npm install → lint → unit tests (Vitest) → build → deploy to S3/CloudFront
  - `dating-app-sp`: build → unit tests → E2E（simulator/emulator or device farm）
  - `dating-app-infra`: Terraform plan → manual approve → Terraform apply (state in S3 + locking via DynamoDB)
- Blue/Green や Canary は最初は不要。API はバージョン運用を想定。

---

## ローカル開発環境

- Backend: ローカルでのユースケーステストは JUnit、LocalStack で DynamoDB を模擬
- Frontend: Storybook（Vue）を導入して UI コンポーネントのレビュー・E2E スタブを容易化
- Mobile: Expo or bare React Native プロジェクト（要検討）、ローカルでのデバッグは Metro

---

## コスト最適化・運用上の工夫

- DynamoDB: オンデマンドで開始、必要に応じてプロビジョニングに切替
- Lambda: 軽量化・タイムアウト短縮、冷却後のレスポンスを許容する M/V 設計
- 開発環境: 必要時のみ稼働（CI での Private infra は自動停止）
- ログ: 詳細ログをデフォルトで取り過ぎない（トラブル時に拡張）

---

## 将来的な拡張案

- 人工知能を使った推薦（マッチングレコメンド）
- Slack/社内チャット連携でペア成立通知
- 高度な分析のために Athena + Glue を利用した週次バッチ
