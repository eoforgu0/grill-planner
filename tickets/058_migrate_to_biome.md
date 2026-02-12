# 058: ESLint + Prettier を Biome に置き換える

## 種別
開発環境改善

## 概要
formatter/linter を ESLint + Prettier（未構成）から Biome に一本化する。

## 作業手順

### 1. Biome のインストール

```bash
npm install --save-dev --save-exact @biomejs/biome
```

### 2. 旧ツールのアンインストール

```bash
npm uninstall eslint @eslint/js eslint-plugin-react-hooks eslint-plugin-react-refresh globals prettier typescript-eslint
```

### 3. 旧設定ファイルの削除

```bash
rm eslint.config.js
```

（.prettierrc 等は現在存在しないため不要）

### 4. biome.json の作成（プロジェクトルート）

```json
{
  "$schema": "https://biomejs.dev/schemas/2.0.0/schema.json",
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true
  },
  "files": {
    "ignoreUnknown": true,
    "ignore": ["dist", "public/data", "public/img"]
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 120
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "double",
      "trailingCommas": "all",
      "semicolons": "always"
    }
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "correctness": {
        "useExhaustiveDependencies": "warn",
        "noUnusedImports": "warn",
        "noUnusedVariables": "warn"
      },
      "style": {
        "useImportType": "error",
        "noNonNullAssertion": "warn"
      },
      "suspicious": {
        "noExplicitAny": "error"
      }
    }
  },
  "organizeImports": {
    "enabled": true
  }
}
```

設定の意図:
- `indentStyle: "space"`, `indentWidth: 2` — 現在のコードスタイルに合わせる
- `lineWidth: 120` — 現在のコードに長い行があるため
- `quoteStyle: "double"` — 現在のコードの大半がダブルクォート
- `trailingCommas: "all"` — diff の見やすさ
- `useExhaustiveDependencies: "warn"` — React Hooks の依存配列チェック
- `noUnusedImports/noUnusedVariables: "warn"` — 不要コードの検出
- `useImportType: "error"` — `import type` の一貫した使用を強制
- `noExplicitAny: "error"` — any 型の禁止（tsconfig の strict と連携）
- `vcs.useIgnoreFile: true` — .gitignore を参照して dist 等を自動除外
- `files.ignore` — データファイルや画像は対象外

### 5. package.json の scripts 更新

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "lint": "biome check .",
    "lint:fix": "biome check --write .",
    "format": "biome format --write .",
    "preview": "vite preview",
    "test": "vitest run"
  }
}
```

### 6. 全ファイルにフォーマット適用

```bash
npx @biomejs/biome check --write .
```

これにより既存コードが Biome のフォーマットに正規化される。差分が大きくなるため、このコミットはフォーマット適用のみ（ロジック変更なし）とすること。

### 7. GitHub Actions の更新

`.github/workflows/deploy.yml` の lint ステップを更新:

```yaml
# 変更前
- run: npm run lint

# 変更後
- run: npx @biomejs/biome ci .
```

`biome ci` は `biome check` と同じだが、CI向けに修正提案の代わりにエラーを返す。

### 8. エディタ設定（任意だが推奨）

VS Code を使用する場合、`.vscode/settings.json` を作成:

```json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.organizeImports.biome": "explicit"
  }
}
```

`.vscode/extensions.json`:
```json
{
  "recommendations": ["biomejs.biome"]
}
```

これらをリポジトリに含めることで、知人が開発に参加する際にも同じ設定が自動適用される。

## 完了条件
- ESLint, Prettier 関連パッケージが package.json から除去されていること
- eslint.config.js が削除されていること
- biome.json が存在し、`npx @biomejs/biome check .` がエラーなしで通ること
- `npm run lint` で Biome が実行されること
- `npm run build` が成功すること（tsc + vite build）
- `npm test` が成功すること
- GitHub Actions の lint ステップが `biome ci` を使用すること
- 全ソースファイルが Biome のフォーマットに正規化されていること
