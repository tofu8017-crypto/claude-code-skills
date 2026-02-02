---
name: dependency-audit
description: >
  プロジェクトの依存関係を包括的に監査する（脆弱性、ライセンス互換性、未使用検出、アップデート計画）。
  Use when user asks to audit dependencies, check licenses, find unused packages,
  plan dependency updates, run npm audit analysis, or clean up package.json.
  Avoid using for installing new dependencies, implementing features,
  fixing runtime bugs, or security review of application code (use security-review instead).
allowed-tools: "Bash(npm:*, npx:*, yarn:*, pnpm:*), Read, Glob, Grep"
metadata:
  author: "user"
  version: "0.1.0"
  category: "dev"
  tags: ["dependencies", "audit", "license", "npm", "security"]
---

# Dependency Audit

プロジェクトの依存関係を包括的に監査し、脆弱性・ライセンス問題・未使用依存・アップデート計画を提供するスキル。

---

## 目的 / Scope

### このスキルが扱う範囲

- **脆弱性スキャン**: npm/yarn audit による CVE 検出と分析
- **ライセンス互換性チェック**: プロジェクトライセンスとの整合性確認
- **未使用依存の検出**: 実際に使用されていないパッケージの特定
- **アップデート計画立案**: 安全な更新順序と破壊的変更の影響評価

### このスキルが扱わない範囲

- **アプリケーションコード自体のセキュリティ**: security-review スキルを使用
- **パフォーマンス分析**: performance-optimizer スキルを使用
- **新規依存のインストール**: impl スキルを使用

---

## いつ起動するか（Trigger）

### ポジティブトリガー

- 「依存関係を監査して」「npm auditして分析して」
- 「ライセンスの互換性を確認して」
- 「使っていない依存を見つけて」「不要なパッケージを削除したい」
- 「依存をアップデートしたい」「outdatedな依存を更新」

### ネガティブトリガー

- 「新しいライブラリをインストールして」→ impl スキル
- 「セキュリティレビューして」→ security-review スキル
- 「アプリが遅い」→ performance-optimizer スキル
- 「エラーが出る」→ verify スキル

---

## 入力として必要な情報（Intake）

### 必須

- **プロジェクトルート**: package.json が存在するディレクトリ

### オプション

- **監査範囲**: all（デフォルト）/ security / license / unused / outdated
- **プロジェクトライセンス**: package.json の `license` から自動取得
- **重要度閾値**: low / moderate / high / critical

### 範囲が不明な場合の確認

```markdown
監査範囲を選択してください:
1. 全項目（推奨）: 脆弱性 + ライセンス + 未使用 + アップデート計画
2. 脆弱性のみ
3. ライセンスのみ
4. 未使用依存のみ
5. アップデート計画のみ
```

---

## 実行手順（Workflow）

### Step 0: 環境検出

#### パッケージマネージャの判定

lockfile で判定:
- `pnpm-lock.yaml` → pnpm
- `yarn.lock` → yarn
- `package-lock.json` → npm
- `bun.lockb` → bun

#### プロジェクト情報

- プロジェクト名、バージョン、ライセンス
- モノレポかどうか（workspaces / pnpm-workspace.yaml）

---

### Step 1: 脆弱性スキャン

#### 実行

```bash
npm audit --json
# or yarn audit --json / pnpm audit --json
```

#### 結果の解析

- 重要度別カウント（critical, high, moderate, low）
- 影響パッケージ、CVE ID、修正バージョン
- 推奨アクション

#### 出力形式

```markdown
## 脆弱性スキャン結果

| 重要度 | 件数 |
|--------|------|
| Critical | X |
| High | X |

### Critical 脆弱性

| パッケージ | CVE | 修正バージョン |
|-----------|-----|--------------|
| example-pkg | CVE-2024-XXXXX | 1.2.4+ |

**修正コマンド**:
npm update example-pkg
```

#### False Positive の除外

- devDependencies のみに影響する脆弱性はランタイムリスク低として分類
- peerDependencies の警告は確認事項として分類

---

### Step 2: ライセンス監査

#### 実行

```bash
npx license-checker --json --production
```

#### 互換性マトリクス照合

[references/license-compatibility.md](references/license-compatibility.md) に基づいて判定。

#### 出力形式

```markdown
## ライセンス監査結果

| 分類 | ライセンス | 件数 |
|------|-----------|------|
| Permissive | MIT, BSD, Apache-2.0, ISC | X |
| Copyleft | GPL, LGPL, AGPL | X |
| 不明 | UNKNOWN | X |

### リスク判定

| パッケージ | ライセンス | 問題 | 対応 |
|-----------|-----------|------|------|
| gpl-pkg | GPL-3.0 | 商用非互換 | 代替パッケージ検討 |
```

#### ライセンス不明パッケージ

1. `npm view {package} license` で確認
2. GitHub リポジトリで LICENSE ファイル確認
3. 不明なまま → ユーザーに報告

---

### Step 3: 未使用依存検出

#### 実行

```bash
npx depcheck --json
```

#### False Positive の除外

以下は要確認として分類（自動除外しない）:
- 動的 import / require
- 設定ファイルでのみ使用（.eslintrc, tailwind.config 等）
- `@types/*` パッケージ
- ビルドツールのプラグイン

#### 出力形式

```markdown
## 未使用依存検出結果

### 削除候補

| パッケージ | サイズ | 推奨 |
|-----------|--------|------|
| lodash | 72 kB | 削除 |
| moment | 289 kB | 削除（date-fns使用中） |

**削除コマンド**:
npm uninstall lodash moment

### 要確認（false positive の可能性）

| パッケージ | 理由 |
|-----------|------|
| tailwindcss | 設定ファイルでのみ使用 |
```

---

### Step 4: アウトデート分析

#### 実行

```bash
npm outdated --json
```

#### Semver 分類

- **Patch** (1.2.3 → 1.2.4): バグ修正のみ、安全に更新可能
- **Minor** (1.2.3 → 1.3.0): 機能追加、後方互換性あり
- **Major** (1.2.3 → 2.0.0): 破壊的変更の可能性

#### Major 更新の調査

- CHANGELOG / GitHub Releases から Breaking Changes を抽出
- Migration Guide の存在確認
- 影響を受けるAPIの特定

#### 出力形式

```markdown
### Patch 更新（安全）

| パッケージ | 現在 | 最新 |
|-----------|------|------|
| axios | 1.5.0 | 1.5.1 |

### Minor 更新（後方互換）

| パッケージ | 現在 | 最新 |
|-----------|------|------|
| next | 14.1.0 | 14.2.0 |

### Major 更新（破壊的変更あり）

| パッケージ | 現在 | 最新 | 主な変更 |
|-----------|------|------|---------|
| react | 17.0.2 | 18.2.0 | Concurrent Mode |
```

---

### Step 5: レポート生成

統合レポートを生成:

```markdown
# 依存関係監査レポート

## エグゼクティブサマリ

| 項目 | 状況 | 重要度 |
|------|------|--------|
| 脆弱性 | Critical: X, High: X | 即対応 |
| ライセンス | 互換性問題: X件 | 確認必要 |
| 未使用依存 | 削除候補: X件 | クリーンアップ推奨 |
| アウトデート | Major更新: X件 | 計画的更新 |
```

---

### Step 6: アクション計画

優先度付きの具体的コマンドリスト:

**P0（即座に対応）**: Critical脆弱性の修正、ライセンス互換性問題
**P1（今週中）**: Patch更新、未使用依存削除
**P2（今月中）**: Minor更新、ライセンス確認
**P3（計画的）**: Major更新、自動更新ツール設定

#### 自動更新ツールの設定（長期運用）

Dependabot / Renovate の設定を提案:
- Patch/Minor の自動マージ
- Major は手動レビュー
- セキュリティアラートの即座対応

---

## 品質ゲート

### 検出品質

- [ ] 全 dependencies/devDependencies をスキャン済み
- [ ] 不明ライセンスを手動確認済み
- [ ] depcheck の誤検出を除外済み
- [ ] Major 更新の影響を評価済み

### 報告品質

- [ ] 優先度（P0-P3）が定義されている
- [ ] コマンドがそのまま実行可能
- [ ] 各アクションのリスクが明記されている
- [ ] ロールバック計画がある

### 定量基準

- 検出漏れ率 5%未満
- ライセンス判定精度 95%以上
- False Positive 率 10%未満

---

## 例

### 例1: 脆弱性検出と修正

**入力**: 「依存関係の脆弱性を調べて」

**出力**: Critical 1件（axios CVE-2023-45857）→ `npm update axios@1.6.0` → 確認

### 例2: 未使用依存クリーンアップ

**入力**: 「使っていないパッケージを見つけて」

**出力**: lodash(72KB) + moment(289KB) が未使用 → `npm uninstall lodash moment` → バンドル361KB削減

### 例3: Major更新計画

**入力**: 「reactを最新にしたい」

**出力**: react 17→18 の段階的計画（準備→更新→検証→ロールバック手順）

### 例4: ライセンス監査

**入力**: 「ライセンスの互換性を確認して。プロジェクトはMIT」

**出力**: GPL-3.0パッケージ検出 → 代替パッケージ提案 or デュアルライセンス確認

---

## 参照

- [ライセンス互換性リファレンス](references/license-compatibility.md)
- [依存更新戦略リファレンス](references/update-strategies.md)

---

## エラー処理

| エラー | 対応 |
|-------|------|
| npm audit が動作しない | npm最新化 + lockfile再生成 |
| depcheck が誤検出 | `.depcheckrc` で除外設定 + 手動確認 |
| ライセンス情報が不明 | npm view + GitHub確認 + ユーザー報告 |
| Major更新後にビルド失敗 | ロールバック → 段階的更新 |
