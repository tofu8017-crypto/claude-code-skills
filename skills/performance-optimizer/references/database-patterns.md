# データベース最適化パターン集

## インデックス設計

### B-tree（デフォルト）

最も汎用的。等値検索、範囲検索に対応:

```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_created ON orders(created_at);

-- 複合インデックス（カラム順序が重要）
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

**複合インデックスの順序**: 等値条件のカラムを先に、範囲条件を後に。

### GIN (Generalized Inverted Index)

配列、JSONB、全文検索向け:

```sql
-- JSONB検索
CREATE INDEX idx_users_metadata ON users USING gin(metadata);

-- 全文検索
CREATE INDEX idx_posts_search ON posts USING gin(to_tsvector('japanese', title || ' ' || content));

-- 配列検索
CREATE INDEX idx_tags ON articles USING gin(tags);
```

### GiST (Generalized Search Tree)

地理データ、範囲型向け:

```sql
-- PostGIS（地理データ）
CREATE INDEX idx_locations_geo ON locations USING gist(geom);

-- 範囲型
CREATE INDEX idx_events_period ON events USING gist(tsrange(start_at, end_at));
```

### インデックス設計の注意点

- 読み取り頻度が高いカラムに作成
- 書き込みが多いテーブルではインデックス数を抑える
- 選択性が低いカラム（boolean等）には不要
- `WHERE` + `ORDER BY` + `JOIN` の条件を考慮

---

## EXPLAIN ANALYZE の読み方

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

### 重要な指標

| 指標 | 意味 | 目安 |
|------|------|------|
| Seq Scan | 全表スキャン | 大きなテーブルでは避ける |
| Index Scan | インデックス使用 | 良好 |
| Index Only Scan | インデックスのみ参照 | 最良 |
| Bitmap Index Scan | 大量ヒット時 | 許容範囲 |
| actual time | 実行時間(ms) | 100ms以下を目指す |
| rows | 処理行数 | 想定と大きく乖離していないか |

### 改善パターン

```
Seq Scan → インデックス追加
Nested Loop (大量行) → JOIN方法の見直し
Sort (大量行) → インデックスでソート済みに
Hash Join (大量メモリ) → work_mem 調整
```

---

## N+1問題の検出と解決

### 検出パターン

```typescript
// NG: N+1問題（100ユーザーで101クエリ）
const users = await supabase.from('users').select('*');
for (const user of users.data) {
  const { data: posts } = await supabase
    .from('posts')
    .select('*')
    .eq('user_id', user.id);
}
```

### 解決: JOIN（Supabase）

```typescript
// OK: 1クエリ
const { data: users } = await supabase
  .from('users')
  .select('*, posts(*)');
```

### 解決: Prisma

```typescript
// OK: include で JOIN
const users = await prisma.user.findMany({
  include: { posts: true },
});
```

### 解決: Drizzle

```typescript
const users = await db.query.users.findMany({
  with: { posts: true },
});
```

---

## JOIN選択

| JOIN | 用途 | パフォーマンス |
|------|------|--------------|
| INNER JOIN | 両方に存在するレコードのみ | 高速（結果セットが小さい） |
| LEFT JOIN | 左テーブル全件 + 右テーブルの一致 | 中程度 |
| EXISTS | 存在チェックのみ | 高速（早期終了可能） |
| IN (サブクエリ) | 小さなリストのマッチング | 小さなリストなら高速 |

```sql
-- EXISTS は COUNT が不要な場合に高速
SELECT * FROM users u
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- IN はリストが小さい場合に使用
SELECT * FROM users WHERE id IN (1, 2, 3, 4, 5);
```

---

## ページネーション最適化

### Offset-based の問題

```sql
-- OFFSET が大きいとパフォーマンスが悪化
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
-- → 10020行をスキャンして最後の20行を返す
```

### Keyset Pagination（推奨）

```sql
-- 前ページの最後のレコードのcreated_atとidを使用
SELECT * FROM posts
WHERE (created_at, id) < ('2024-01-15T10:00:00', 'uuid-xxx')
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- → インデックスを使って効率的にスキャン
```

**Supabase での実装**:

```typescript
const { data } = await supabase
  .from('posts')
  .select('*')
  .order('created_at', { ascending: false })
  .lt('created_at', lastCreatedAt)
  .limit(20);
```

---

## Supabase 固有の最適化

### RLS（Row Level Security）のパフォーマンス

```sql
-- NG: 遅いRLSポリシー（サブクエリが重い）
CREATE POLICY "users_policy" ON posts
  USING (user_id IN (SELECT id FROM users WHERE organization_id = auth.jwt() ->> 'org_id'));

-- OK: 直接比較（高速）
CREATE POLICY "users_policy" ON posts
  USING (user_id = auth.uid());
```

### リアルタイム購読の最適化

```typescript
// NG: テーブル全体を購読
supabase.channel('*').on('postgres_changes', { event: '*', schema: 'public', table: 'messages' }, handleChange);

// OK: フィルタリングして購読
supabase.channel('user-messages').on('postgres_changes', {
  event: 'INSERT',
  schema: 'public',
  table: 'messages',
  filter: `user_id=eq.${userId}`,
}, handleChange);
```

### 接続プーリング

- Supavisor を使用（Supabase デフォルト）
- `pgbouncer` モードでトランザクションプーリング
- 接続URL に `?pgbouncer=true` を追加
