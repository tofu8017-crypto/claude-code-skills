---
name: api-design-generator
description: >
  RESTful API設計とOpenAPI 3.x仕様書を生成する。
  Use when user asks to design APIs, create API documentation,
  generate OpenAPI specs, review API endpoint design, or document existing APIs.
  Avoid using for GraphQL schema design, WebSocket protocol design,
  or general code documentation unrelated to APIs.
allowed-tools: "Bash(npx:*), Read, Write, Glob, Grep, Edit"
metadata:
  author: "user"
  version: "0.1.0"
  category: "dev"
  tags: ["api", "openapi", "rest", "design"]
---

# API Design Generator

RESTful API設計とOpenAPI 3.x仕様書を自動生成するスキル。

---

## 目的 / Scope

### 成果

- RESTful設計原則に準拠したAPI設計
- OpenAPI 3.x YAML仕様書の生成
- 既存コードからのAPI仕様書抽出
- API設計のレビューと改善提案

### 対象外

- GraphQL スキーマ設計
- WebSocket プロトコル設計
- gRPC / Protocol Buffers 定義
- API実装コードの生成（設計のみ）

---

## いつ起動するか（Trigger）

### 代表的なフレーズ

- 「APIを設計して」「エンドポイントを設計したい」「REST APIの構成を考えて」
- 「OpenAPI仕様書を作って」「Swagger定義を作成して」
- 「このAPIのドキュメントを生成して」「エンドポイントを文書化したい」
- 「API設計をレビューして」「REST設計を改善したい」

### ネガティブトリガー（起動しない）

- 「GraphQL APIを設計して」→ GraphQL専用ツール
- 「WebSocketサーバーを作って」→ WebSocket実装
- 「APIを実装して」→ impl スキル
- 「コードのコメントを書いて」→ ドキュメントスキル

---

## 入力として必要な情報（Intake）

### 必須情報

| 情報 | 例 | 不足時の確認質問 |
|------|-----|-----------------|
| APIの目的 | ユーザー管理、商品カタログ | 「このAPIは何を管理するものですか？」 |
| 対象リソース | User, Product, Order | 「主要なリソースは何ですか？」 |
| 操作の種類 | CRUD, 検索, フィルタリング | 「各リソースに対してどんな操作が必要ですか？」 |

### 任意情報（デフォルト値あり）

| 情報 | デフォルト値 | 用途 |
|------|-------------|------|
| 認証方式 | Bearer Token | securitySchemes 定義 |
| バージョニング | URL パス（/v1/） | API バージョン管理 |
| ページネーション | offset-based | 一覧取得の方式 |
| エラーフォーマット | RFC 7807 Problem Details | エラーレスポンス統一 |

### 既存コードから生成する場合の確認

1. 「プロジェクトのフレームワークは何ですか？（Express, Next.js など）」
2. 「ルート定義があるファイルを教えてください」
3. 「既存の型定義はありますか？（TypeScriptの場合）」

---

## 実行手順（Workflow）

### Step 0: 前提チェック

プロジェクト構造を確認し、既存APIの有無を判定。

1. `package.json` の有無確認 → フレームワーク判定
2. `openapi.yaml`, `swagger.yaml` の存在確認 → 既存仕様があれば「追記」か「新規作成」か確認
3. `src/routes/`, `app/api/`, `pages/api/` の存在確認 → 既存コードの有無

**使用ツール**: `Glob`: `**/package.json`, `**/openapi.yaml`, `**/routes/**`

| 状況 | アクション |
|------|-----------|
| 既存 OpenAPI あり | 「既存仕様に追記しますか？」確認 |
| 既存コードあり | 「既存コードから生成しますか？」確認 |
| なし | 新規設計フローへ |

---

### Step 1: 要件整理

#### 1-1. リソースの特定

ユーザーからの情報をもとに、APIで扱うリソースを整理：

| リソース名 | 属性例 | 関連リソース |
|-----------|-------|-------------|
| User | id, name, email | Profile, Order |
| Product | id, name, price | Category, Review |

#### 1-2. アクターと操作の整理

| アクター | リソース | 操作 |
|---------|---------|------|
| 一般ユーザー | Product | 一覧取得、詳細取得 |
| 認証済みユーザー | Order | 作成、自分のOrder取得 |
| 管理者 | User | 全操作（CRUD） |

---

### Step 2: リソース設計

RESTful命名規約に準拠したURL設計：

**命名規約チェック**:
- リソース名は複数形（`/users`, `/products`）
- kebab-case を使用（`/order-items`）
- ネストは最大2階層まで（`/users/{id}/orders` まで）
- 動詞を含めない（`/getUsers` は NG）

**リレーションの表現**:
- 親子関係が明確 → サブリソース（`/users/{id}/orders`）
- 多対多、複雑な条件 → クエリパラメータ（`/orders?user_id={id}`）

**参照**: [references/rest-design-principles.md](references/rest-design-principles.md)

---

### Step 3: エンドポイント設計

#### HTTPメソッドとCRUDのマッピング

| 操作 | HTTPメソッド | URL | ステータスコード |
|------|-------------|-----|----------------|
| 一覧取得 | GET | `/users` | 200 OK |
| 詳細取得 | GET | `/users/{id}` | 200 OK |
| 作成 | POST | `/users` | 201 Created |
| 更新（全体） | PUT | `/users/{id}` | 200 OK |
| 更新（部分） | PATCH | `/users/{id}` | 200 OK |
| 削除 | DELETE | `/users/{id}` | 204 No Content |

#### ステータスコードの選定

| カテゴリ | コード | 用途 |
|---------|-------|------|
| 成功 | 200 OK | 取得・更新成功 |
| 成功 | 201 Created | 作成成功（Locationヘッダー付与） |
| 成功 | 204 No Content | 削除成功 |
| クライアントエラー | 400 Bad Request | バリデーションエラー |
| クライアントエラー | 401 Unauthorized | 認証が必要 |
| クライアントエラー | 403 Forbidden | 権限不足 |
| クライアントエラー | 404 Not Found | リソースが存在しない |
| クライアントエラー | 409 Conflict | 重複など |
| サーバーエラー | 500 Internal Server Error | サーバー内部エラー |

#### 検索・フィルタリング

- フィルタ: `?role=admin`, `?category=electronics`
- ソート: `?sort=created_at`, `?sort=-name`（降順）
- ページネーション（offset-based）: `?offset=20&limit=10`
- ページネーション（cursor-based）: `?cursor=abc123&limit=10`

**参照**: [references/openapi-patterns.md](references/openapi-patterns.md)

---

### Step 4: スキーマ設計

#### リクエストボディスキーマ

```yaml
CreateUserRequest:
  type: object
  required:
    - name
    - email
  properties:
    name:
      type: string
      minLength: 1
      maxLength: 100
    email:
      type: string
      format: email
    role:
      type: string
      enum: [user, admin]
      default: user
```

#### レスポンスボディスキーマ

```yaml
UserResponse:
  type: object
  properties:
    id:
      type: string
      format: uuid
    name:
      type: string
    email:
      type: string
      format: email
    created_at:
      type: string
      format: date-time
```

#### エラーレスポンススキーマ（RFC 7807 統一）

```yaml
ProblemDetails:
  type: object
  properties:
    type:
      type: string
      format: uri
    title:
      type: string
    status:
      type: integer
    detail:
      type: string
    instance:
      type: string
      format: uri
```

---

### Step 5: OpenAPI YAML生成

#### 基本構造

```yaml
openapi: 3.0.3
info:
  title: [APIタイトル]
  version: 1.0.0
  description: [API説明]
servers:
  - url: https://api.example.com/v1
    description: Production
```

#### paths セクション

エンドポイントごとに summary, operationId, tags, parameters, requestBody, responses, security を定義。

#### components セクション

- `schemas`: リクエスト/レスポンスのデータ定義
- `securitySchemes`: 認証方式の定義（bearerAuth, apiKey 等）
- `responses`: 共通エラーレスポンス（BadRequest, Unauthorized, NotFound）

#### ファイル出力

| 状況 | 出力先 |
|------|-------|
| プロジェクトルート直下 | `openapi.yaml` |
| docs/ ディレクトリあり | `docs/openapi.yaml` |
| api/ ディレクトリあり | `api/openapi.yaml` |

---

### Step 6: バリデーション

```bash
npx @apidevtools/swagger-cli validate openapi.yaml
```

**チェック項目**:
- YAML構文エラーなし
- OpenAPI 3.x 仕様準拠
- $ref 参照が正しい
- 必須フィールドが存在

---

## 品質ゲート（Quality Checklist）

### RESTful設計原則

- [ ] リソース名は複数形（`/users`, `/products`）
- [ ] URL に動詞を含まない
- [ ] kebab-case を使用
- [ ] ネストは最大2階層
- [ ] HTTPメソッドが適切

### ステータスコード

- [ ] 成功時のステータスコードが適切（200, 201, 204）
- [ ] エラー時のステータスコードが適切（400, 401, 403, 404, 500）
- [ ] 201 Created 時に Location ヘッダーを返す

### 認証・認可

- [ ] 認証が必要なエンドポイントに `security` を定義
- [ ] securitySchemes を定義

### ページネーション

- [ ] 一覧取得エンドポイントにページネーションパラメータ
- [ ] レスポンスにページネーション情報

### エラーレスポンス

- [ ] 全エラーが統一フォーマット（RFC 7807 推奨）
- [ ] 400, 401, 403, 404, 500 のレスポンススキーマを定義

### ドキュメント品質

- [ ] 各エンドポイントに `summary` と `description`
- [ ] タグでエンドポイントを分類

---

## 例

### UC1: 新規API設計

**入力**: 「ブログAPIを設計して。記事の作成、取得、更新、削除ができるようにしたい。」

**出力骨子**:

```yaml
openapi: 3.0.3
info:
  title: Blog API
  version: 1.0.0
paths:
  /posts:
    get:
      summary: 記事一覧取得
      responses:
        '200':
          description: 成功
    post:
      summary: 記事作成
      responses:
        '201':
          description: 作成成功
  /posts/{id}:
    get:
      summary: 記事詳細取得
    put:
      summary: 記事更新
    delete:
      summary: 記事削除
      responses:
        '204':
          description: 削除成功
```

### UC2: 既存コードからAPI仕様書生成

**入力**: 「このExpressアプリのAPIドキュメントを作って」

**処理**: Expressのルート定義ファイルを検索 → ルート解析 → 型定義があれば抽出 → OpenAPI YAML生成

### UC3: API設計レビュー

**入力**: 「このAPI設計をレビューして」（既存の openapi.yaml を提示）

**出力例**:
- RESTful命名規約違反の指摘と修正案
- ステータスコードの適切性チェック
- エラーレスポンス・ページネーションの不足指摘

---

## 参照

- [OpenAPIパターン集](references/openapi-patterns.md)
- [RESTful設計原則](references/rest-design-principles.md)

---

## エラー処理

| エラー | 原因 | 対応 |
|-------|------|------|
| バリデーションエラー | 生成したYAMLが不正 | Step 6 に戻り修正 |
| 既存ファイルと衝突 | openapi.yaml が既に存在 | 上書き確認または別名で保存 |
| フレームワーク不明 | コードから生成時にルート構造不明 | ユーザーに手動指定を依頼 |
