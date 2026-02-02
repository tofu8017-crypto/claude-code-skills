# バンドルサイズ分析・削減パターン集

## バンドル分析ツール

### @next/bundle-analyzer

```bash
npm install --save-dev @next/bundle-analyzer
```

```js
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});
module.exports = withBundleAnalyzer({ /* config */ });
```

```bash
ANALYZE=true npm run build
```

### webpack-bundle-analyzer

```bash
npx webpack-bundle-analyzer .next/static/chunks/stats.json
```

### Vite

```bash
npx vite-bundle-visualizer
```

---

## Tree-shaking が効かないパターン

### CommonJS モジュール

```javascript
// NG: CommonJS は Tree-shaking 不可
const _ = require('lodash');

// OK: ESM なら Tree-shaking 可能
import { uniq } from 'lodash-es';
```

### 副作用のあるモジュール

```javascript
// package.json で sideEffects を確認
{
  "sideEffects": false  // Tree-shaking 安全
}

// sideEffects: true の場合、バンドラは安全に削除できない
```

### バレルファイル（index.ts）の問題

```typescript
// NG: バレルファイルから1つだけ使用しても全体がバンドルされる場合がある
import { Button } from '@/components';

// OK: 直接インポート
import { Button } from '@/components/Button';
```

---

## Code Splitting 戦略

### Route-based（ページ単位）

Next.js App Router では自動:
```
app/
  page.tsx        → メインバンドル
  dashboard/
    page.tsx      → dashboardバンドル（遅延ロード）
  settings/
    page.tsx      → settingsバンドル（遅延ロード）
```

### Component-based（コンポーネント単位）

```tsx
import dynamic from 'next/dynamic';

// 重いコンポーネントのみ遅延ロード
const RichTextEditor = dynamic(() => import('./RichTextEditor'), {
  ssr: false,
  loading: () => <EditorSkeleton />,
});

// 条件付きロード
const AdminPanel = dynamic(() => import('./AdminPanel'));
function Page({ isAdmin }) {
  return isAdmin ? <AdminPanel /> : null;
}
```

### Library-based（ライブラリ単位）

```tsx
// 重いライブラリを必要時のみロード
async function generatePDF() {
  const { jsPDF } = await import('jspdf');
  const doc = new jsPDF();
  // ...
}
```

---

## 重い依存の軽量代替

| 重いライブラリ | サイズ | 軽量代替 | サイズ | 削減量 |
|--------------|--------|---------|--------|--------|
| moment | 67KB | date-fns | 12KB | -55KB |
| moment | 67KB | dayjs | 2KB | -65KB |
| lodash | 24KB | lodash-es (個別import) | 2KB | -22KB |
| lodash | 24KB | ネイティブJS | 0KB | -24KB |
| axios | 13KB | fetch (ネイティブ) | 0KB | -13KB |
| classnames | 1KB | clsx | 0.5KB | -0.5KB |
| uuid | 3KB | crypto.randomUUID() | 0KB | -3KB |

### lodash → ネイティブJS

```javascript
// lodash
import { uniq, flatten, groupBy } from 'lodash-es';

// ネイティブJS
const uniq = (arr) => [...new Set(arr)];
const flatten = (arr) => arr.flat();
const groupBy = (arr, key) =>
  arr.reduce((acc, item) => {
    (acc[item[key]] ??= []).push(item);
    return acc;
  }, {});
```

---

## Dynamic Import の実践パターン

### イベント駆動ロード

```tsx
function ShareButton() {
  const handleShare = async () => {
    const { share } = await import('@/lib/share');
    share({ title: 'My Page', url: window.location.href });
  };
  return <button onClick={handleShare}>共有</button>;
}
```

### 条件付きロード

```tsx
function Analytics() {
  useEffect(() => {
    if (process.env.NODE_ENV === 'production') {
      import('@/lib/analytics').then(({ init }) => init());
    }
  }, []);
  return null;
}
```

### Intersection Observer でのロード

```tsx
function LazySection() {
  const [Component, setComponent] = useState(null);
  const ref = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        import('./HeavySection').then((mod) => setComponent(() => mod.default));
        observer.disconnect();
      }
    });
    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  return <div ref={ref}>{Component ? <Component /> : <Skeleton />}</div>;
}
```

---

## next.config.js の最適化設定

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  // 画像最適化
  images: {
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200],
  },

  // 不要なパッケージをバンドルから除外
  experimental: {
    optimizePackageImports: [
      'lucide-react',
      '@heroicons/react',
      'date-fns',
      'lodash-es',
    ],
  },

  // Webpack 設定
  webpack: (config, { isServer }) => {
    // moment のロケールを除外（moment使用時）
    config.plugins.push(
      new webpack.IgnorePlugin({
        resourceRegExp: /^\.\/locale$/,
        contextRegExp: /moment$/,
      })
    );
    return config;
  },

  // コンパイラ最適化
  compiler: {
    removeConsole: process.env.NODE_ENV === 'production',
  },
};
```

---

## バンドルサイズの目安

| 指標 | 良好 | 注意 | 危険 |
|------|------|------|------|
| First Load JS (per route) | 100KB以下 | 100-200KB | 200KB超 |
| 共有バンドル | 80KB以下 | 80-150KB | 150KB超 |
| 個別チャンク | 50KB以下 | 50-100KB | 100KB超 |
| 合計JS（gzip後） | 200KB以下 | 200-400KB | 400KB超 |
