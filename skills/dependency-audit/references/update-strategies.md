# 依存更新戦略リファレンス

## Semver の読み方と影響範囲

### バージョン記法

```
MAJOR.MINOR.PATCH
 2   .  3  .  1
```

| 部分 | 変更時の意味 | 影響 |
|------|-------------|------|
| MAJOR | 破壊的変更（非互換API変更） | コード修正が必要な場合あり |
| MINOR | 機能追加（後方互換性あり） | コード修正不要 |
| PATCH | バグ修正（後方互換性あり） | コード修正不要 |

### package.json の記法

| 記法 | 意味 | 例 |
|------|------|-----|
| `1.2.3` | 完全一致 | 1.2.3 のみ |
| `^1.2.3` | MINOR/PATCH更新を許可 | 1.2.3 〜 1.x.x |
| `~1.2.3` | PATCH更新のみ許可 | 1.2.3 〜 1.2.x |
| `>=1.2.3` | 以上 | 1.2.3 以上すべて |

---

## 安全な更新順序

### 基本原則: Patch → Minor → Major

```
Phase 1: Patch 更新（リスク: 最低）
  npm update  # semver範囲内の更新
  npm test    # テスト確認

Phase 2: Minor 更新（リスク: 低）
  npm install package@^latest-minor
  npm test

Phase 3: Major 更新（リスク: 中〜高）
  # 1つずつ更新
  npm install package@latest
  npm test
```

### 依存ツリーの順序: Leaf → Root

```
依存ツリー:
  my-app
  ├── react (root dependency)
  │   └── react-dom
  ├── @tanstack/react-query
  │   └── react (peer)
  └── next
      └── react (peer)

更新順序:
1. Leaf: react-dom（他に依存がない）
2. Middle: @tanstack/react-query
3. Root: react, next（多くのパッケージが依存）
```

---

## 破壊的変更の調査方法

### 1. CHANGELOG の確認

```bash
# GitHub Releases を確認
gh release list -R owner/repo --limit 10

# CHANGELOG.md を直接確認
curl -s https://raw.githubusercontent.com/owner/repo/main/CHANGELOG.md | head -100
```

### 2. Migration Guide の検索

一般的な場所:
- `UPGRADING.md`
- `MIGRATION.md`
- `docs/migration/`
- 公式ドキュメントの `/migration` ページ

### 3. Breaking Changes の抽出

```bash
# CHANGELOG から BREAKING を検索
curl -s https://raw.githubusercontent.com/owner/repo/main/CHANGELOG.md | grep -A 5 -i "breaking"
```

### 4. TypeScript 型変更の確認

```bash
# 型エラーで影響範囲を確認
npx tsc --noEmit 2>&1 | head -50
```

---

## ロールバック計画

### 事前準備

```bash
# 更新前にブランチを作成
git checkout -b deps/update-package

# lockfile をコミット
git add package-lock.json
git commit -m "chore: snapshot before dependency update"
```

### ロールバック手順

```bash
# 方法1: git から復元
git checkout main -- package.json package-lock.json
npm install

# 方法2: 特定バージョンに戻す
npm install package@1.2.3
```

### lockfile からの完全復元

```bash
# lockfile が正しい状態なら
npm ci  # clean install (lockfile通りにインストール)
```

---

## lockfile 管理のベストプラクティス

### 基本ルール

| ルール | 理由 |
|-------|------|
| lockfile を必ずコミットする | ビルドの再現性を保証 |
| `npm ci` を CI で使用する | lockfile 通りにインストール |
| `npm install` は開発時のみ | lockfile を更新する |
| lockfile の手動編集を避ける | 整合性が壊れる |

### lockfile の競合解決

```bash
# lockfile の競合時
git checkout --theirs package-lock.json
npm install
# → package.json に基づいて lockfile を再生成

# または
rm package-lock.json
npm install
```

---

## 自動更新ツール

### Dependabot

`.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Asia/Tokyo"
    open-pull-requests-limit: 10
    groups:
      # Patch 更新をグループ化
      patch-updates:
        update-types:
          - "patch"
      # TypeScript 型定義をグループ化
      types:
        patterns:
          - "@types/*"
    labels:
      - "dependencies"
    reviewers:
      - "team/frontend"
```

### Renovate

`renovate.json`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:base",
    ":timezone(Asia/Tokyo)",
    "schedule:weekly"
  ],
  "packageRules": [
    {
      "description": "Auto-merge patch updates",
      "matchUpdateTypes": ["patch", "pin", "digest"],
      "automerge": true
    },
    {
      "description": "Auto-merge dev dependency minor updates",
      "matchDepTypes": ["devDependencies"],
      "matchUpdateTypes": ["minor"],
      "automerge": true
    },
    {
      "description": "Group TypeScript related updates",
      "matchPackagePatterns": ["^@types/", "typescript"],
      "groupName": "TypeScript"
    },
    {
      "description": "Group ESLint related updates",
      "matchPackagePatterns": ["eslint"],
      "groupName": "ESLint"
    }
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "schedule": ["at any time"]
  }
}
```

### Dependabot vs Renovate

| 機能 | Dependabot | Renovate |
|------|-----------|----------|
| グループ化 | 基本的 | 高度（正規表現、カスタムルール） |
| 自動マージ | GitHub Actions 連携が必要 | 組み込み |
| モノレポ | 基本対応 | 高度な対応 |
| スケジュール | 日/週/月 | cron 式で柔軟 |
| PR サイズ | 1パッケージ1PR（グループ化可能） | 柔軟にグループ化 |
| セットアップ | GitHub ネイティブ | 追加設定が必要 |

---

## モノレポでの依存管理

### ワークスペース間の依存統一

```bash
# pnpm の場合
pnpm up --recursive  # 全ワークスペースの依存を更新

# npm workspaces の場合
npm update --workspaces

# 特定パッケージのみ
npm update react --workspace=packages/frontend
```

### バージョン統一の確認

```bash
# 全ワークスペースで使用されているバージョンを確認
pnpm why react --recursive
```

### 依存の巻き上げ（hoisting）

```yaml
# pnpm-workspace.yaml
packages:
  - 'packages/*'

# .npmrc (pnpm)
shamefully-hoist=true  # 注意: 互換性のため
```
