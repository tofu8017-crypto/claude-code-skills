# RESTful 設計原則リファレンス

## リソース命名規約

### 基本ルール

| ルール | 良い例 | 悪い例 |
|-------|--------|--------|
| 複数形を使用 | `/users` | `/user` |
| kebab-case | `/order-items` | `/orderItems`, `/order_items` |
| 名詞を使用（動詞禁止） | `/users` | `/getUsers`, `/createUser` |
| ネスト最大2階層 | `/users/{id}/orders` | `/users/{id}/orders/{oid}/items/{iid}` |
| 小文字のみ | `/users` | `/Users`, `/USERS` |
| 末尾スラッシュなし | `/users` | `/users/` |

### サブリソース vs クエリパラメータ

| パターン | URL | いつ使う |
|---------|-----|---------|
| サブリソース | `/users/{id}/orders` | 明確な親子関係 |
| クエリパラメータ | `/orders?user_id={id}` | 検索・フィルタリング |

**判断基準**:
- リソースBがリソースAなしに存在しない → サブリソース
- リソースBが独立して存在し得る → クエリパラメータ
- 多対多の関係 → クエリパラメータ

---

## HTTPメソッドとCRUDの対応

| 操作 | HTTPメソッド | URL | ボディ | 応答 |
|------|-------------|-----|--------|------|
| 一覧取得 | GET | `/resources` | なし | 200 + リスト |
| 詳細取得 | GET | `/resources/{id}` | なし | 200 + 単体 |
| 作成 | POST | `/resources` | あり | 201 + 作成物 + Location |
| 全体更新 | PUT | `/resources/{id}` | あり（全フィールド） | 200 + 更新物 |
| 部分更新 | PATCH | `/resources/{id}` | あり（変更フィールドのみ） | 200 + 更新物 |
| 削除 | DELETE | `/resources/{id}` | なし | 204 |

### PUT vs PATCH

- **PUT**: リソース全体を置き換える。送信しないフィールドはnull/デフォルトになる
- **PATCH**: 指定したフィールドのみ更新。送信しないフィールドは変更されない

### アクション系操作

CRUD に当てはまらない操作:

```
POST /orders/{id}/cancel     # 注文キャンセル
POST /users/{id}/verify      # メール認証
POST /payments/{id}/refund   # 返金
```

---

## ステータスコード選択ガイド

### 成功系 (2xx)

| コード | 用途 | メソッド |
|-------|------|---------|
| 200 OK | 取得・更新成功 | GET, PUT, PATCH |
| 201 Created | 作成成功 | POST |
| 202 Accepted | 非同期処理を受付 | POST |
| 204 No Content | 削除成功（ボディなし） | DELETE |

### クライアントエラー (4xx)

| コード | 用途 |
|-------|------|
| 400 Bad Request | バリデーションエラー、不正なJSON |
| 401 Unauthorized | 認証が必要（トークンなし/無効） |
| 403 Forbidden | 認証済みだが権限不足 |
| 404 Not Found | リソースが存在しない |
| 405 Method Not Allowed | HTTPメソッドが許可されていない |
| 409 Conflict | 重複作成、楽観ロック失敗 |
| 415 Unsupported Media Type | Content-Typeが不正 |
| 422 Unprocessable Entity | バリデーション（構文は正しいが意味が不正） |
| 429 Too Many Requests | レート制限超過 |

### サーバーエラー (5xx)

| コード | 用途 |
|-------|------|
| 500 Internal Server Error | サーバー内部エラー |
| 502 Bad Gateway | 上流サーバーからの不正応答 |
| 503 Service Unavailable | メンテナンス中 |

### 400 vs 422 の使い分け

- **400**: JSONの構文エラー、必須パラメータ欠落
- **422**: JSON構文は正しいがビジネスロジック違反（例: 開始日が終了日より後）

---

## バージョニング戦略

### URL パスバージョニング（推奨）

```
https://api.example.com/v1/users
https://api.example.com/v2/users
```

**メリット**: 明示的、キャッシュ可能、デバッグ容易
**デメリット**: URL変更、クライアント更新必要

### ヘッダーバージョニング

```
Accept: application/vnd.example.v1+json
```

**メリット**: URLがクリーン
**デメリット**: ブラウザでのテスト困難

### クエリパラメータ

```
https://api.example.com/users?version=1
```

**メリット**: 実装が簡単
**デメリット**: キャッシュ問題

### バージョンアップのガイドライン

- 破壊的変更がある場合のみメジャーバージョンを上げる
- 新フィールド追加は破壊的変更に含めない
- 旧バージョンは最低6ヶ月サポート
- 廃止予定は `Sunset` ヘッダーで通知

---

## HATEOAS の適用判断

### HATEOAS とは

レスポンスに次のアクションへのリンクを含める:

```json
{
  "id": "123",
  "name": "John",
  "links": {
    "self": "/users/123",
    "orders": "/users/123/orders",
    "update": "/users/123",
    "delete": "/users/123"
  }
}
```

### 適用すべきケース

- 公開API（外部開発者向け）
- 状態遷移が複雑なワークフロー
- 権限によって利用可能な操作が変わる場合

### 適用不要なケース

- 内部API（フロントエンド専用）
- シンプルなCRUD
- パフォーマンスが最優先

---

## べき等性と安全性

### 定義

| 性質 | 意味 |
|------|------|
| 安全（Safe） | サーバーの状態を変更しない |
| べき等（Idempotent） | 同じリクエストを何度実行しても結果が同じ |

### メソッド別の性質

| メソッド | 安全 | べき等 |
|---------|------|--------|
| GET | Yes | Yes |
| HEAD | Yes | Yes |
| OPTIONS | Yes | Yes |
| PUT | No | Yes |
| DELETE | No | Yes |
| POST | No | No |
| PATCH | No | No |

### べき等性の実装パターン

**PUT**: 同じボディで何度呼んでも同じ結果

```
PUT /users/123
{ "name": "John", "email": "john@example.com" }
→ 何度実行しても user 123 は同じ状態
```

**DELETE**: 2回目以降は 404 でもよい（結果的にべき等）

```
DELETE /users/123 → 204 (1回目)
DELETE /users/123 → 404 (2回目、既に削除済み)
```

**POST のべき等化**: Idempotency-Key ヘッダーを使用

```
POST /payments
Idempotency-Key: unique-request-id-123
{ "amount": 1000 }
→ 同じキーで再送信しても二重決済にならない
```
