# OpenAPI 3.x パターン集

## 基本構造テンプレ

```yaml
openapi: 3.0.3
info:
  title: API Title
  version: 1.0.0
  description: API description
  contact:
    name: API Support
    email: support@example.com
servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://api-staging.example.com/v1
    description: Staging
paths: {}
components:
  schemas: {}
  securitySchemes: {}
  responses: {}
```

---

## 認証スキーム

### Bearer Token (JWT)

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT Bearer token

security:
  - bearerAuth: []
```

### API Key

```yaml
components:
  securitySchemes:
    apiKeyHeader:
      type: apiKey
      in: header
      name: X-API-Key
    apiKeyQuery:
      type: apiKey
      in: query
      name: api_key
```

### OAuth 2.0

```yaml
components:
  securitySchemes:
    oauth2:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://auth.example.com/authorize
          tokenUrl: https://auth.example.com/token
          scopes:
            read:users: Read user information
            write:users: Modify user information
            admin: Full admin access
```

### エンドポイント別セキュリティ

```yaml
paths:
  /public:
    get:
      security: []  # 認証不要
  /private:
    get:
      security:
        - bearerAuth: []  # 認証必要
  /admin:
    get:
      security:
        - bearerAuth: []
        - oauth2: [admin]  # 管理者権限必要
```

---

## ページネーションパターン

### Offset-based

```yaml
paths:
  /users:
    get:
      parameters:
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
            minimum: 0
          description: Number of items to skip
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
            minimum: 1
            maximum: 100
          description: Number of items to return
      responses:
        '200':
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  pagination:
                    $ref: '#/components/schemas/OffsetPagination'

components:
  schemas:
    OffsetPagination:
      type: object
      properties:
        total:
          type: integer
          description: Total number of items
        offset:
          type: integer
        limit:
          type: integer
        has_more:
          type: boolean
```

### Cursor-based（大量データ向け）

```yaml
parameters:
  - name: cursor
    in: query
    schema:
      type: string
    description: Cursor for pagination (opaque string)
  - name: limit
    in: query
    schema:
      type: integer
      default: 20
      maximum: 100

components:
  schemas:
    CursorPagination:
      type: object
      properties:
        next_cursor:
          type: string
          nullable: true
          description: Next page cursor (null if last page)
        has_more:
          type: boolean
```

**選択基準**:
- offset-based: 総件数表示が必要、ページジャンプが必要
- cursor-based: 大量データ、リアルタイム更新、安定したパフォーマンス

---

## エラーレスポンス（RFC 7807 Problem Details）

```yaml
components:
  schemas:
    ProblemDetails:
      type: object
      properties:
        type:
          type: string
          format: uri
          description: A URI reference identifying the problem type
          example: https://api.example.com/errors/validation
        title:
          type: string
          description: A short human-readable summary
          example: Validation Error
        status:
          type: integer
          description: HTTP status code
          example: 400
        detail:
          type: string
          description: A human-readable explanation
          example: The email field is required
        instance:
          type: string
          format: uri
          description: A URI identifying the specific occurrence
        errors:
          type: array
          items:
            type: object
            properties:
              field:
                type: string
              message:
                type: string
          description: Field-level validation errors

  responses:
    BadRequest:
      description: Validation error
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'
          example:
            type: https://api.example.com/errors/validation
            title: Validation Error
            status: 400
            detail: One or more fields are invalid
            errors:
              - field: email
                message: Invalid email format

    Unauthorized:
      description: Authentication required
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'

    Forbidden:
      description: Insufficient permissions
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'

    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'

    Conflict:
      description: Resource conflict (e.g., duplicate)
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'
```

---

## ファイルアップロード

### Single file

```yaml
paths:
  /users/{id}/avatar:
    put:
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              properties:
                file:
                  type: string
                  format: binary
                  description: Image file (JPEG, PNG, max 5MB)
            encoding:
              file:
                contentType: image/jpeg, image/png
      responses:
        '200':
          description: Upload successful
          content:
            application/json:
              schema:
                type: object
                properties:
                  url:
                    type: string
                    format: uri
```

### Multiple files

```yaml
requestBody:
  content:
    multipart/form-data:
      schema:
        type: object
        properties:
          files:
            type: array
            items:
              type: string
              format: binary
          description:
            type: string
```

---

## レート制限ヘッダ

```yaml
components:
  headers:
    X-RateLimit-Limit:
      description: Maximum requests per time window
      schema:
        type: integer
    X-RateLimit-Remaining:
      description: Remaining requests in current window
      schema:
        type: integer
    X-RateLimit-Reset:
      description: UTC epoch timestamp when the window resets
      schema:
        type: integer

  responses:
    TooManyRequests:
      description: Rate limit exceeded
      headers:
        X-RateLimit-Limit:
          $ref: '#/components/headers/X-RateLimit-Limit'
        X-RateLimit-Remaining:
          $ref: '#/components/headers/X-RateLimit-Remaining'
        X-RateLimit-Reset:
          $ref: '#/components/headers/X-RateLimit-Reset'
        Retry-After:
          description: Seconds to wait before retrying
          schema:
            type: integer
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'
```

---

## ソート・フィルタリング

```yaml
parameters:
  - name: sort
    in: query
    schema:
      type: string
      example: -created_at
    description: |
      Sort field. Prefix with - for descending.
      Examples: created_at, -updated_at, name
  - name: filter[status]
    in: query
    schema:
      type: string
      enum: [active, inactive, pending]
    description: Filter by status
  - name: filter[created_after]
    in: query
    schema:
      type: string
      format: date-time
    description: Filter items created after this date
  - name: q
    in: query
    schema:
      type: string
    description: Full-text search query
```
