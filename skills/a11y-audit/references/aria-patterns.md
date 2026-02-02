# ARIA パターンリファレンス

## 基本原則: "No ARIA is better than bad ARIA"

### ARIA使用の判断フロー

1. ネイティブHTML要素で実現可能か？ → **Yes → ARIAは不要**
2. ネイティブでは不可能な複雑なウィジェットか？ → **Yes → ARIAを使用**
3. ARIAを使う場合、実装と一致しているか？ → **必ず確認**

| ネイティブHTML | ARIA代替（不要） |
|--------------|----------------|
| `<button>` | `<div role="button">` |
| `<a href>` | `<span role="link">` |
| `<input type="checkbox">` | `<div role="checkbox">` |
| `<nav>` | `<div role="navigation">` |
| `<main>` | `<div role="main">` |

---

## ランドマークロール

```html
<header role="banner">
  <nav role="navigation" aria-label="メインナビゲーション">...</nav>
</header>

<main role="main">
  <section aria-labelledby="section-title">
    <h2 id="section-title">セクションタイトル</h2>
    ...
  </section>
</main>

<aside role="complementary" aria-label="サイドバー">...</aside>

<footer role="contentinfo">...</footer>
```

**注意**: HTML5セマンティック要素を使う場合、role属性は冗長（ただし古いブラウザ対応では追加する）。

---

## ウィジェットパターン

### Dialog（モーダル）

```tsx
<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="dialog-title"
  aria-describedby="dialog-desc"
>
  <h2 id="dialog-title">確認</h2>
  <p id="dialog-desc">この操作を実行しますか？</p>
  <button>OK</button>
  <button>キャンセル</button>
</div>
```

**必須要件**:
- `role="dialog"` + `aria-modal="true"`
- `aria-labelledby` でタイトルを参照
- フォーカスをダイアログ内にトラップ
- Escapeキーで閉じる
- 閉じた後、開いた要素にフォーカスを戻す

### Tabs

```tsx
<div role="tablist" aria-label="設定タブ">
  <button role="tab" aria-selected="true" aria-controls="panel-1" id="tab-1">
    一般
  </button>
  <button role="tab" aria-selected="false" aria-controls="panel-2" id="tab-2" tabindex="-1">
    セキュリティ
  </button>
</div>

<div role="tabpanel" id="panel-1" aria-labelledby="tab-1">
  一般設定のコンテンツ
</div>
<div role="tabpanel" id="panel-2" aria-labelledby="tab-2" hidden>
  セキュリティ設定のコンテンツ
</div>
```

**キーボード操作**:
- 左右矢印キーでタブ切り替え
- Home/Endで最初/最後のタブ
- 選択中のタブのみ `tabindex="0"`、他は `tabindex="-1"`

### Accordion

```tsx
<div>
  <h3>
    <button
      aria-expanded="true"
      aria-controls="section-1"
    >
      セクション1
    </button>
  </h3>
  <div id="section-1" role="region" aria-labelledby="section-1-btn">
    コンテンツ
  </div>
</div>
```

### Menu

```tsx
<button aria-haspopup="true" aria-expanded="false" aria-controls="menu-1">
  メニュー
</button>
<ul role="menu" id="menu-1">
  <li role="menuitem">プロフィール</li>
  <li role="menuitem">設定</li>
  <li role="separator"></li>
  <li role="menuitem">ログアウト</li>
</ul>
```

**キーボード操作**:
- Enter/Spaceでメニュー開閉
- 上下矢印キーで項目移動
- Escapeで閉じる

### Combobox（オートコンプリート）

```tsx
<label htmlFor="search">検索</label>
<input
  id="search"
  role="combobox"
  aria-expanded="true"
  aria-autocomplete="list"
  aria-controls="search-listbox"
  aria-activedescendant="option-2"
/>
<ul role="listbox" id="search-listbox">
  <li role="option" id="option-1">結果1</li>
  <li role="option" id="option-2" aria-selected="true">結果2</li>
  <li role="option" id="option-3">結果3</li>
</ul>
```

### Tooltip

```tsx
<button aria-describedby="tooltip-1">
  ヘルプ
</button>
<div role="tooltip" id="tooltip-1">
  この機能の詳細説明
</div>
```

---

## ライブリージョン

動的に更新されるコンテンツをスクリーンリーダーに通知:

### aria-live

| 値 | 動作 | 用途 |
|---|------|------|
| `polite` | 現在の読み上げ終了後に通知 | 検索結果件数、ステータス更新 |
| `assertive` | 即座に通知（現在の読み上げを中断） | エラー、警告 |
| `off` | 通知しない（デフォルト） | - |

```tsx
// 検索結果件数の通知
<div aria-live="polite" aria-atomic="true">
  {results.length}件の結果が見つかりました
</div>

// エラー通知
<div role="alert">  {/* role="alert" = aria-live="assertive" と同等 */}
  メールアドレスが無効です
</div>

// ステータス通知
<div role="status">  {/* role="status" = aria-live="polite" と同等 */}
  保存しました
</div>
```

### aria-atomic

- `true`: 変更時に領域全体を読み上げ
- `false`: 変更された部分のみ読み上げ

### aria-relevant

通知する変更の種類:
- `additions`: 追加されたノード
- `removals`: 削除されたノード
- `text`: テキスト変更
- `all`: すべて
- デフォルト: `additions text`

---

## フォームパターン

### ラベルと説明

```tsx
<label htmlFor="password">パスワード</label>
<input
  id="password"
  type="password"
  aria-describedby="password-help"
/>
<div id="password-help">8文字以上、大文字小文字を含む</div>
```

### エラーメッセージ

```tsx
<label htmlFor="email">メールアドレス</label>
<input
  id="email"
  type="email"
  aria-invalid="true"
  aria-errormessage="email-error"
/>
<div id="email-error" role="alert">
  有効なメールアドレスを入力してください
</div>
```

### 必須フィールド

```tsx
<label htmlFor="name">
  名前 <span aria-hidden="true">*</span>
</label>
<input id="name" aria-required="true" />
```

---

## 状態管理

| 属性 | 値 | 用途 |
|------|---|------|
| `aria-expanded` | true/false | アコーディオン、ドロップダウンの開閉 |
| `aria-selected` | true/false | タブ、リスト項目の選択 |
| `aria-checked` | true/false/mixed | チェックボックスの状態 |
| `aria-pressed` | true/false | トグルボタンの状態 |
| `aria-disabled` | true | 無効化された要素 |
| `aria-busy` | true/false | 読み込み中 |
| `aria-hidden` | true | 視覚的には見えるがスクリーンリーダーから隠す |
| `aria-current` | page/step/location/date/true | 現在のページ/ステップ |

### 実装と一致させる

```tsx
// NG: aria-expanded="true" なのに内容が非表示
<button aria-expanded="true">メニュー</button>
<ul style={{ display: 'none' }}>...</ul>  // 矛盾!

// OK: 状態と実装が一致
<button aria-expanded={isOpen}>{isOpen ? '閉じる' : '開く'}</button>
{isOpen && <ul>...</ul>}
```
