---
name: a11y-audit
description: >
  Webアプリケーションのアクセシビリティ（WCAG 2.2 AA準拠）を監査し、具体的な修正を提供する。
  Use when user asks to check accessibility, audit a11y, verify WCAG compliance,
  add ARIA attributes, improve screen reader support, fix keyboard navigation,
  or make components accessible.
  Avoid using for visual design changes, CSS styling unrelated to accessibility,
  SEO optimization, or general code review without accessibility focus.
allowed-tools: "Bash(npx:*), Read, Glob, Grep, Edit"
metadata:
  author: "user"
  version: "0.1.0"
  category: "dev"
  tags: ["accessibility", "a11y", "wcag", "aria"]
---

# a11y-audit

WebアプリケーションのアクセシビリティをWCAG 2.2 AA基準で監査し、具体的な修正案を提供するスキル。

---

## 目的 / Scope

- WCAG 2.2 AA準拠の監査（自動チェック + 手動レビュー）
- 実装可能な修正提案（抽象的な指摘ではなく具体的なコード修正）
- 教育的フィードバック（なぜその修正が必要かの説明）

### 対象

- React/Vue/Svelte等のコンポーネント
- HTML/JSX/TSXファイル
- フォーム、ダイアログ、メニュー等のインタラクティブ要素
- ページ全体の構造（ランドマーク、見出し階層）

### 対象外

- PDFやネイティブアプリのアクセシビリティ
- WCAG AAA レベル（要求がある場合のみ）
- 視覚デザイン自体の改善（色コントラスト以外）

---

## いつ起動するか（Trigger）

### ポジティブトリガー

| トリガー | 例 |
|---------|-----|
| 明示的な監査依頼 | 「アクセシビリティをチェックして」「a11y監査して」 |
| WCAG準拠確認 | 「WCAGに準拠しているか確認」 |
| ARIA関連 | 「ARIA属性を追加して」「スクリーンリーダー対応して」 |
| キーボード操作 | 「キーボードナビゲーションを改善」 |
| フォームa11y | 「フォームのエラーメッセージをアクセシブルに」 |

### ネガティブトリガー

- 「色を変更して」「デザインを改善」（a11y無関係）
- 「SEOを最適化」
- 「パフォーマンスを改善」
- 一般的なコードレビュー（a11y観点が明示されていない）

---

## 入力として必要な情報（Intake）

### 必須

1. **対象ファイル/コンポーネント**: ユーザー指定 or 最近変更されたファイルから推測
2. **監査スコープ**: コンポーネント単位 or ページ全体

### 不足時の確認

「どのファイル/コンポーネントを監査しますか？」
1. 最近変更したファイル
2. 特定のファイルを指定
3. プロジェクト全体

---

## 実行手順（Workflow）

### Step 0: 環境チェック

1. `package.json` からフレームワーク特定（React, Next.js, Vue, Svelte）
2. a11yツールの有無確認（axe-core, eslint-plugin-jsx-a11y, jest-axe）
3. ツールがない場合: インストールを提案

### Step 1: 対象範囲の特定

```
├── 単一コンポーネント → そのファイルのみ読み込み
├── ページ全体 → メインページ + 使用コンポーネントを特定
└── プロジェクト全体 → src/components/**/*.{tsx,jsx,vue,svelte}
```

### Step 2: 自動チェック

#### axe-core（推奨）

```bash
npx axe http://localhost:3000/page-to-test --tags wcag2aa
```

#### eslint-plugin-jsx-a11y

```bash
npx eslint src/components/Dialog.tsx --plugin jsx-a11y
```

#### ツールなしの場合

手動レビュー（Step 3）のみ実行。インストール推奨を通知。

### Step 3: 手動コードレビュー

[references/wcag-checklist.md](references/wcag-checklist.md) を参照。

#### 3.1 セマンティックHTML

| チェック項目 | 良い例 | 悪い例 |
|-------------|--------|--------|
| 適切なHTML要素 | `<button>` | `<div onClick>` |
| 見出し階層 | h1 → h2 → h3 | h1 → h3（スキップ） |
| リスト | `<ul><li>` | `<div>` + CSS bullets |
| ランドマーク | `<nav>`, `<main>` | `<div id="nav">` |

#### 3.2 テキスト代替

| 対象 | 必須属性 |
|------|---------|
| `<img>` | `alt` |
| 装飾画像 | `alt=""` |
| `<svg>` | `aria-label` or `<title>` |
| アイコンボタン | `aria-label` |
| フォーム要素 | `<label>` or `aria-label` |

#### 3.3 色コントラスト

WCAG AA基準:
- 通常テキスト: 4.5:1 以上
- 大きなテキスト（18pt以上 or 14pt太字）: 3:1 以上

#### 3.4 フォーカス管理

| チェック項目 | 要件 |
|-------------|------|
| フォーカス可視性 | `:focus-visible` スタイル必須 |
| フォーカス順序 | DOM順序 = Tab順序 |
| モーダル内トラップ | フォーカスがモーダル外に出ない |
| 閉じた後の復帰 | 開いた要素に戻る |

#### 3.5 ARIA使用の適切性

[references/aria-patterns.md](references/aria-patterns.md) を参照。

**重要原則**: "No ARIA is better than bad ARIA"

### Step 4: WCAG 2.2 AA基準との照合

| 原則 | 主要基準 |
|------|---------|
| 1. 知覚可能 | テキスト代替、適応可能、判別可能（色コントラスト） |
| 2. 操作可能 | キーボード操作、ナビゲーション、入力モダリティ |
| 3. 理解可能 | 読み取り可能、予測可能、入力支援（エラー通知） |
| 4. 堅牢 | 互換性（正しいHTML、name/role/value） |

### Step 5: 問題の重要度分類

| 重要度 | 基準 | 例 |
|--------|------|-----|
| Critical | 主要機能が使用不可 | フォーム送信ボタンにキーボードアクセス不可 |
| Major | 一部ユーザーが困難 | 画像にalt属性なし、コントラスト不足 |
| Minor | 改善推奨 | 見出し階層の軽微な問題 |

### Step 6: 修正コード生成

各問題に対して即座に適用できる修正コードを提示。

#### 修正パターン例

**クリック可能なdiv → button**:
```tsx
// Before
<div onClick={handleClick}>Click me</div>

// After
<button onClick={handleClick}>Click me</button>
```

**アイコンボタンにaria-label**:
```tsx
// Before
<button onClick={onClose}><XIcon /></button>

// After
<button onClick={onClose} aria-label="閉じる">
  <XIcon aria-hidden="true" />
</button>
```

**フォームラベルの関連付け**:
```tsx
// Before
<span>メールアドレス</span>
<input type="email" name="email" />

// After
<label htmlFor="email">メールアドレス</label>
<input type="email" id="email" name="email" />
```

**エラーメッセージの通知**:
```tsx
<input
  type="email"
  id="email"
  aria-invalid={!!error}
  aria-describedby={error ? "email-error" : undefined}
/>
{error && (
  <div id="email-error" role="alert">{error}</div>
)}
```

**モーダルのアクセシビリティ**:
```tsx
<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="modal-title"
>
  <h2 id="modal-title">{title}</h2>
  {/* フォーカストラップのロジックも実装 */}
</div>
```

### Step 7: 修正後の検証方法提示

#### 自動テスト

```bash
npx axe http://localhost:3000/page --tags wcag2aa
npx eslint src/components/Dialog.tsx
```

#### 手動テスト

- **キーボード操作**: Tabキーのみで全操作可能か
- **スクリーンリーダー**: VoiceOver (Cmd+F5) or NVDA で読み上げ確認

#### CIへの組み込み（推奨）

```typescript
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

test('Dialog should have no a11y violations', async () => {
  const { container } = render(<Dialog />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

---

## 品質ゲート

### 必須基準

- [ ] セマンティックHTML: div/span の乱用がない
- [ ] ARIA使用の適切性: 過剰ARIAがない
- [ ] キーボード操作可能: 全インタラクティブ要素がTabアクセス可能
- [ ] 色だけに依存しない: エラー等で色以外の手がかりがある
- [ ] フォーカス可視: `:focus-visible` スタイルが適用されている

### 推奨基準

- [ ] テキスト代替: 全画像/アイコンに意味のあるalt/aria-label
- [ ] 見出し階層: h1 → h2 → h3 の順序が正しい
- [ ] ランドマーク: `<nav>`, `<main>`, `<aside>` を使用
- [ ] エラー通知: `role="alert"` または `aria-live` で動的通知
- [ ] フォーカス管理: モーダル内フォーカストラップ、閉じた後の復帰

---

## 例

### 例1: フォームの監査

**入力**: 「サインアップフォームのa11yをチェックして」

**出力**:
```markdown
## 検出された問題（3件）

### Critical（1件）
#1: パスワード入力にラベルがない（WCAG 1.3.1）

### Major（2件）
#2: エラーメッセージが動的に通知されない（WCAG 4.1.3）
#3: 送信ボタンのフォーカススタイルがない（WCAG 2.4.7）
```

### 例2: ダイアログの監査

**入力**: 「モーダルダイアログをアクセシブルにして」

**出力**:
- role="dialog" + aria-modal の追加
- フォーカストラップの実装
- Escapeキーで閉じる機能
- 背景スクロール無効化

### 例3: プロジェクト全体の監査

**入力**: 「プロジェクト全体のa11yスコアを出して」

**出力**:
```markdown
## スコア: 68/100 (C評価)

| カテゴリ | スコア | 問題数 |
|---------|--------|--------|
| 知覚可能 | 75/100 | 8件 |
| 操作可能 | 60/100 | 12件 |
| 理解可能 | 70/100 | 6件 |
| 堅牢 | 65/100 | 9件 |

Critical: 5件、Major: 15件、Minor: 15件
```

---

## 参照

- [WCAG 2.2 AA チェックリスト](references/wcag-checklist.md)
- [ARIA パターンリファレンス](references/aria-patterns.md)

---

## エラー処理

| エラー | 対応 |
|-------|------|
| 対象ファイルが見つからない | 別ファイル指定 or プロジェクト全体から検索 |
| フレームワークが不明 | ユーザーに確認して適切な修正パターンを選択 |
| 自動チェックツールのエラー | 手動レビューで続行、サーバー起動を案内 |
