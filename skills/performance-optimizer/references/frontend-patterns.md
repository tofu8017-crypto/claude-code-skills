# React/Next.js パフォーマンスパターン集

## React.memo / useMemo / useCallback

### React.memo の使用判断

**使うべき場合**:
- 親の再レンダリングで不要に再レンダリングされるコンポーネント
- レンダリングコストが高いコンポーネント（大きなリスト、複雑なUI）
- 同じpropsで頻繁に再レンダリングされる

**使わなくてよい場合**:
- レンダリングが軽いコンポーネント（メモ化コスト > レンダリングコスト）
- propsが毎回変わるコンポーネント

```tsx
const ExpensiveList = React.memo(function ExpensiveList({ items, onSelect }) {
  return items.map(item => <Item key={item.id} item={item} onSelect={onSelect} />);
});
```

### useMemo

高コストな計算結果をキャッシュ:

```tsx
const sortedItems = useMemo(() => {
  return items.slice().sort((a, b) => a.name.localeCompare(b.name));
}, [items]);
```

### useCallback

関数参照を安定化（React.memo と組み合わせ）:

```tsx
const handleSelect = useCallback((id: string) => {
  setSelected(id);
}, []);
```

---

## 仮想スクロール

大量リスト（1000件以上）のレンダリング:

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }) {
  const parentRef = useRef(null);
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });

  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              transform: `translateY(${virtualItem.start}px)`,
              height: `${virtualItem.size}px`,
            }}
          >
            {items[virtualItem.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## 画像最適化

### next/image

```tsx
import Image from 'next/image';

// LCP対象の画像（Above the fold）
<Image src="/hero.jpg" width={1200} height={600} priority alt="Hero" />

// 通常の画像（lazy loading）
<Image
  src="/product.jpg"
  width={400}
  height={300}
  alt="Product"
  placeholder="blur"
  blurDataURL="data:image/..."
/>
```

### WebP/AVIF

next.config.js で自動変換:
```js
module.exports = {
  images: {
    formats: ['image/avif', 'image/webp'],
  },
};
```

---

## Dynamic Import / React.lazy

### Next.js dynamic

```tsx
import dynamic from 'next/dynamic';

const HeavyChart = dynamic(() => import('@/components/HeavyChart'), {
  ssr: false,
  loading: () => <ChartSkeleton />,
});
```

### React.lazy + Suspense

```tsx
const LazyComponent = React.lazy(() => import('./HeavyComponent'));

<Suspense fallback={<Loading />}>
  <LazyComponent />
</Suspense>
```

---

## Server Components vs Client Components

### 判断基準

| 条件 | Server Component | Client Component |
|------|-----------------|-----------------|
| データフェッチ | Yes | No (use hook) |
| 状態管理 (useState) | No | Yes |
| イベントハンドラ (onClick) | No | Yes |
| ブラウザAPI (window, document) | No | Yes |
| 静的コンテンツ表示 | Yes | 不要 |

### パターン

```tsx
// Server Component (default in App Router)
async function ProductPage({ params }) {
  const product = await getProduct(params.id); // サーバーでフェッチ
  return (
    <div>
      <h1>{product.name}</h1>
      <AddToCartButton productId={product.id} /> {/* Client Component */}
    </div>
  );
}

// Client Component
'use client';
function AddToCartButton({ productId }) {
  const [loading, setLoading] = useState(false);
  return <button onClick={() => addToCart(productId)}>カートに追加</button>;
}
```

---

## Suspense境界の設計

```tsx
// ページレベルのSuspense
<Suspense fallback={<PageSkeleton />}>
  <Header />
  <Suspense fallback={<ContentSkeleton />}>
    <MainContent />
  </Suspense>
  <Suspense fallback={<SidebarSkeleton />}>
    <Sidebar />
  </Suspense>
</Suspense>
```

**設計原則**:
- 独立してロードできる部分を別のSuspense境界で囲む
- Above the fold のコンテンツは最優先でロード
- Skeleton UIでレイアウトシフトを防止

---

## Core Web Vitals 改善パターン

### LCP (Largest Contentful Paint) - 目標: 2.5秒以下

| 原因 | 対策 |
|------|------|
| 大きな画像 | next/image + priority属性 |
| レンダリングブロックCSS | Critical CSS inline + 残りは非同期 |
| サーバー応答遅い | SSG/ISR、CDN活用 |
| JS実行が長い | Code Splitting、Dynamic Import |

### INP (Interaction to Next Paint) - 目標: 200ms以下

| 原因 | 対策 |
|------|------|
| 重い状態更新 | useDeferredValue, startTransition |
| 同期的なレンダリング | React.memo, 仮想スクロール |
| 長い計算 | Web Worker への移動 |

```tsx
import { startTransition } from 'react';

function SearchBar() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const handleChange = (e) => {
    setQuery(e.target.value); // 即座に更新
    startTransition(() => {
      setResults(search(e.target.value)); // 低優先度で更新
    });
  };
}
```

### CLS (Cumulative Layout Shift) - 目標: 0.1以下

| 原因 | 対策 |
|------|------|
| 画像サイズ未指定 | width/height属性を必ず指定 |
| フォント読み込み | `font-display: swap` + preload |
| 動的コンテンツ挿入 | min-height で領域を確保 |
| 広告/埋め込み | aspect-ratio で領域を確保 |
