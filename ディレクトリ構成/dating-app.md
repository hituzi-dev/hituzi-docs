# dating-app: ディレクトリ構成（リポジトリ分割を含む）

このドキュメントは、プロジェクトを複数リポジトリに分ける前提（SP / FE / BE / インフラ）で、各リポジトリ内の推奨ディレクトリ構成とテスト・アーキテクチャ設計（クリーンアーキテクチャ + アトミックデザイン）を具体的に示します。

---

## リポジトリ命名例

- dating-app-sp
- dating-app-fe
- dating-app-be
- dating-app-infra

---

## 共通方針

- クリーンアーキテクチャ（Presentation / UseCase / Domain / Infrastructure）を基準にレイヤー分け
- アトミックデザイン（atoms/molecules/organisms/templates/pages）を FE（SP 含む）で採用
- テスト: Unit (Jest/Vitest/JUnit)、Integration (LocalStack)、E2E (Playwright/Maestro)
- Storybook を FE リポジトリに導入して UI コンポーネントの文書化と E2E のスタブを容易化

---

## `dating-app-be` (Java + AWS Lambda)

推奨ディレクトリ構成 (Maven/Gradle ベース):

- src/
  - main/
    - java/
      - com.example.datingapp/
        - adapter/            # Infrastructure（DynamoDB, API Gateway などの実装）
          - controller/       # Lambda ハンドラ層（API 入力の変換）
          - repository/       # DynamoDB 実装
          - gateway/          # 外部 API 連携クライアント
        - application/        # UseCase 層
          - service/          # ユースケース実装
          - dto/              # 入出力 DTO
        - domain/             # ドメインモデル
          - model/
          - repository/       # Domain インターフェース
          - service/          # ドメインサービス
        - config/
    - resources/
  - test/
    - java/                # Unit / Integration tests

- build.gradle / pom.xml
- infra/                   # SAM or Terraform のテンプレート（軽微な infra を含める場合）
- docs/                    # API spec / アーキテクチャ図
- localstack/              # LocalStack 用の設定/スクリプト

テスト配置:
- Unit: src/test/java (Mockito, JUnit)
- Integration: src/test/java/integration (LocalStack を起動して DynamoDB などとテスト)

設計ポイント:
- UseCase はインターフェースを通じて Domain とやり取りし、Adapter 層外から Domain に直接依存しない
- DTO は Controller 層でマッピングし、Domain モデルはユースケース/ドメイン層で扱う

---

## `dating-app-fe` (Vue)

推奨ディレクトリ:

- src/
  - components/
    - atoms/
    - molecules/
    - organisms/
    - templates/
  - pages/
  - layouts/
  - store/ (pinia or vuex)
  - services/ (API clients)
  - composables/ (共通のロジック)
  - styles/
  - tests/
    - unit/
    - e2e/
- public/
- .storybook/
- vitest.config.ts
- playwright.config.ts (or vitest E2E)

テスト:
- Unit: Vitest + Vue Testing Library
- E2E: Playwright または Vitest E2E
- Storybook を使って UI ドキュメント化

アトミック設計の例:
- atoms/Button.vue, atoms/Input.vue
- molecules/UserCard.vue (atoms を組み合わせる)
- organisms/PairList.vue
- templates/HomeTemplate.vue
- pages/HomePage.vue

---

## `dating-app-sp` (React Native)

推奨ディレクトリ:

- src/
  - components/
    - atoms/
    - molecules/
    - organisms/
  - screens/ (pages 相当)
  - navigation/
  - store/ (redux or recoil/pinia-like)
  - services/ (API client)
  - hooks/
  - storybook/
  - tests/
    - unit/
    - e2e/

テスト:
- Unit: Jest + React Native Testing Library
- E2E: Maestro（デバイス / エミュレータ）
- Storybook を導入してコンポーネント確認

---

## `dating-app-infra` (Terraform)

推奨構成:

- modules/
  - vpc/
  - dynamodb/
  - lambda/
  - cognito/
  - api_gateway/
- envs/
  - dev/
  - staging/
  - prod/
- scripts/
- README.md

状態管理:
- Terraform state -> S3
- Locking -> DynamoDB

---

## CI/CD（リポジトリ別ワークフロー）

- 共通: PR -> Lint/Unit Test -> Build -> Integration/E2E (optional) -> Deploy (manual approval for infra)
- `dating-app-be`: Build(JDK) -> Unit -> Integration(LocalStack) -> Deploy
- `dating-app-fe`: npm ci -> lint -> unit -> storybook build -> e2e -> deploy to S3
- `dating-app-sp`: yarn install -> unit -> build -> upload artifacts / device farm

---

## クリーンアーキテクチャ → 実装例（FE の場合）

- Presentation (pages, components) -> UseCase (composables/services) -> Domain (interfaces/models) -> Infrastructure (services/api clients)

例: ペア作成フロー
- pages/PairPage.vue -> composable/useCreatePair.ts (UseCase) -> domain/Pair.ts -> services/apiClient.ts (Infrastructure)

---

## ドキュメント運用ルール（簡易）

- 各リポジトリの `README.md` にセットアップ手順を必ず明記
- API の契約は OpenAPI で管理し、`dating-app-be` の docs/ に配置
- Storybook は `dating-app-fe` / `dating-app-sp` に配置し、PR でレビューを必須化
