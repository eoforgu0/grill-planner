# 083: Biome lint を error 昇格 + コミット前 lint ルール化

## 種別
環境整備

## 背景

チケット081対応後、未使用インポートが残ったまま push → GitHub Actions の `biome ci` でエラーとなりデプロイ失敗。
`biome ci` は warning もエラー扱いするが、ローカルの `biome check`（`npm run lint`）は warning を exit 0 で通すため、コミット前に実行しても見逃しやすい。

## 修正内容

### 1. biome.json — warn を error に昇格

```jsonc
"correctness": {
  "useExhaustiveDependencies": "warn",  // これはwarnのまま（依存配列は判断が必要な場合があるため）
  "noUnusedImports": "error",           // warn → error
  "noUnusedVariables": "error"          // warn → error
}
```

これにより `npm run lint`（`biome check .`）でも未使用インポート・変数がエラーとなり、exit 1 で失敗する。

### 2. package.json — ci 用 lint スクリプト追加

```json
"scripts": {
  ...
  "lint:ci": "biome ci .",
}
```

`biome ci` は `biome check` と同等だが warning もエラー扱いにする。
GitHub Actions の deploy.yml で既に `npx @biomejs/biome ci .` を使用しているが、`npm run lint:ci` でもローカルから同じチェックを実行できるようにする。

### 3. 運用ルール

**コミット前に `npm run lint` を実行すること。**

手順:
```bash
npm run lint      # エラーがあれば修正
npm run lint:fix  # 自動修正可能なものは自動修正
git add .
git commit -m "..."
```

上記の `noUnusedImports: "error"` 昇格により、`npm run lint` で未使用インポートがエラーとして報告されるため、コミット前に気づける。

※ huskyやlint-stagedなどのgit hook による自動化は、個人開発でのセットアップコストに見合わないため導入しない。

## 完了条件
- `biome.json` で `noUnusedImports` と `noUnusedVariables` が `"error"` になっていること
- `useExhaustiveDependencies` は `"warn"` のままであること
- 未使用インポートがある状態で `npm run lint` を実行すると exit 1 で失敗すること
- `package.json` に `lint:ci` スクリプトが追加されていること
