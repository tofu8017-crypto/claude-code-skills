---
name: performance-optimizer
description: >
  フロントエンド、データベース、バンドルサイズのパフォーマンスを分析・最適化する。
  Use when user mentions slow rendering, query optimization, bundle size reduction,
  performance profiling, React/Next.js optimization, SQL tuning, or Lighthouse scores.
  Avoid using for functional bug fixes, UI design changes, feature implementation,
  or security vulnerability fixes unrelated to performance.
allowed-tools: "Bash(npx:*, node:*, npm run:*), Read, Glob, Grep"
metadata:
  author: "user"
  version: "0.1.0"
  category: "dev"
  tags: ["performance", "optimization", "frontend", "database", "bundle"]
---

# Performance Optimizer

フロントエンド、データベース、バンドルサイズのパフォーマンスボトルネックを特定し、具体的な改善策を提示するスキル。

---

## 目的 / Scope

| 領域 | 対象 |
|------|------|
| フロントエンド | React/Next.jsレンダリング、Core Web Vitals、画像最適化 |
| データベース | SQLクエリ、N+1問題、インデックス設計、RLS最適化 |
| バンドルサイズ | JavaScriptバンドル、Code Splitting、Tree-shaking |

### 対象外

- セキュリティ脆弱性の修正（パフォーマンスに影響しない場合）
- 機能バグの修正
- UI/UXデザインの改善
- インフラのスケーリング（アプリケーションコード外）

---

## いつ起動するか（Trigger）

### ポジティブトリガー

- **フロントエンド**: 「表示が遅い」「レンダリングを最適化」「Lighthouse」「Core Web Vitals」「LCP/INP/CLS」
- **データベース**: 「クエリが遅い」「SQLを最適化」「N+1問題」「EXPLAIN ANALYZE」「Supabaseのパフォーマンス」
- **バンドル**: 「バンドルが大きい」「ビルドサイズ」「First Load JS」

### ネガティブトリガー

- パフォーマンスと無関係なバグ修正（例: 「ボタンが押せない」）
- 機能追加（例: 「新しいフォームを作りたい」）
- セキュリティ問題（例: 「XSS対策を追加」）

---

## 入力として必要な情報（Intake）

### 必須

- **問題の症状**: 何が遅いか（ページ表示、API応答、ビルド時間等）
- **対象ファイル/機能**: どの部分を最適化したいか

### 自動収集

- フレームワーク（Next.js/Vite/CRA等）
- ビルドツール（Webpack/Turbopack/esbuild等）
- データベース（PostgreSQL/Supabase等）

---

## 実行手順（Workflow）

### Step 0: 環境検出

プロジェクトの技術スタックを自動検出。

```bash
# package.json からフレームワークを特定
cat package.json | grep -E '"(next|react|vite|astro)"'
```

出力:
```markdown
| 項目 | 検出内容 |
|------|---------|
| フレームワーク | Next.js 14 (App Router) |
| ビルドツール | Turbopack |
| データベース | Supabase (PostgreSQL) |
```

### Step 1: パフォーマンス問題の分類

| 症状 | 分類 | 次のStep |
|------|------|---------|
| ページ表示が遅い | フロントエンド | Step 2-A |
| API応答が遅い | データベース | Step 2-B |
| ビルドが重い | バンドルサイズ | Step 2-C |
| リスト表示が重い | フロントエンド + DB | Step 2-A, 2-B |

### Step 2-A: フロントエンド計測

#### Lighthouse 分析

```bash
npx lighthouse https://your-site.com --view --output=html
```

**確認ポイント**:
- **LCP** (Largest Contentful Paint): 目標 2.5秒以下
- **INP** (Interaction to Next Paint): 目標 200ms以下
- **CLS** (Cumulative Layout Shift): 目標 0.1以下

#### React DevTools Profiler

レンダリング時間が長いコンポーネント、不必要な再レンダリングを特定。

### Step 2-B: データベース計測

#### EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'test@example.com';
```

**確認ポイント**: Seq Scan の有無、実行時間、Index Scan の使用状況

#### N+1問題の検出

```javascript
// NG: N+1 問題
const users = await db.users.findMany();
for (const user of users) {
  const posts = await db.posts.findMany({ where: { userId: user.id } });
}

// OK: JOINで解決
const users = await db.users.findMany({ include: { posts: true } });
```

### Step 2-C: バンドル分析

```bash
# Next.js
ANALYZE=true npm run build

# Vite
npx vite-bundle-visualizer
```

### Step 3: ボトルネック特定

| 優先度 | 基準 | 例 |
|--------|------|-----|
| Critical | ユーザー体験に直接影響 | LCP 4秒超、クエリ 1秒超 |
| High | 一部のユーザーに影響 | INP 300ms超、バンドル 500KB超 |
| Medium | 改善の余地あり | レンダリング 50ms超、N+1 |
| Low | 最適化候補 | 微細な再レンダリング |

### Step 4: 最適化提案

各提案に以下を含める:
- **具体的なコード変更**（Before/After）
- **期待効果の概算**（ms, KB, render count等）
- **副作用リスク**
- **優先度**

#### フロントエンド最適化パターン

**React.memo / useMemo の適用**:
```tsx
// Before: 毎回再レンダリング
function UserCard({ user, onSelect }) { ... }

// After: propsが変わらなければスキップ
const UserCard = React.memo(function UserCard({ user, onSelect }) { ... });
```

**Dynamic Import**:
```tsx
const HeavyChart = dynamic(() => import('@/components/HeavyChart'), {
  ssr: false,
  loading: () => <div>読み込み中...</div>
});
```

**next/image**:
```tsx
<Image src="/hero.png" width={1200} height={600} priority alt="Hero" />
```

#### データベース最適化パターン

**N+1解決（Supabase）**:
```typescript
// Before: 1 + N回のクエリ
const { data: users } = await supabase.from('users').select('*');

// After: 1回のクエリ
const { data: users } = await supabase.from('users').select('*, posts(*)');
```

**インデックス追加**:
```sql
CREATE INDEX idx_users_email ON users(email);
```

#### バンドルサイズ削減パターン

**ライブラリ置換**: moment.js(67KB) → date-fns(12KB)、lodash(24KB) → lodash-es(2KB)

### Step 5: 実装支援

実装の優先順位:
1. **Quick Win**（即効性 + 低リスク）: インデックス追加、next/image、Dynamic import
2. **High Impact**（効果大 + 中リスク）: N+1解決、React.memo、ライブラリ置換
3. **Long Term**（効果大 + 高リスク）: アーキテクチャ変更、Server Components 移行

### Step 6: 効果検証

改善前後の比較方法を提示:

```markdown
| 指標 | 改善前 | 改善後 | 改善率 |
|------|--------|--------|--------|
| LCP | 4.2秒 | 2.1秒 | -50% |
| API応答 | 3.2秒 | 0.3秒 | -91% |
| First Load JS | 285 KB | 185 KB | -35% |
```

---

## 品質ゲート

- [ ] すべての提案に定量メトリクスが含まれる
- [ ] 期待効果の概算が記載されている
- [ ] 副作用リスクが明記されている
- [ ] Critical/High/Medium/Low で優先度付けされている

---

## 例

### 例1: フロントエンド最適化

**入力**: 「Next.jsのページ表示が遅い。Lighthouseスコアが40点」

**出力**:
1. 画像最適化不足 [Critical] → next/image 適用 → LCP 2.5秒短縮
2. バンドルサイズ過大 [High] → Dynamic import → 100KB削減

### 例2: データベース最適化

**入力**: 「Supabaseのクエリが遅い。ユーザー一覧取得に3秒」

**出力**:
1. N+1問題 [Critical] → select に join 追加 → 101回→1回、3.2秒→0.3秒

### 例3: バンドルサイズ削減

**入力**: 「First Load JSが大きい」

**出力**:
1. moment.js [High] → date-fns に置換 → 55KB削減
2. lodash [Medium] → lodash-es に変更 → 22KB削減

---

## 参照

- [フロントエンドパターン集](references/frontend-patterns.md)
- [データベース最適化パターン集](references/database-patterns.md)
- [バンドル分析パターン集](references/bundle-analysis.md)
