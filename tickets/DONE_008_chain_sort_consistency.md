# 008: validateSlotChain のソート方向の不具合リスク

## 種別
バグ（予防的）

## 問題
006 のチェーン検証は「降順ソートした湧き配列を、降順ソートした撃破配列に対して先頭から1対1マッチング」する方式だが、ソート方向を間違えると完全に壊れる。

現在の validation.ts 内には降順ソートが多数あり、一部は呼び出し側で、一部は関数内でソートしている。統一されていない。

## 修正箇所
- `src/utils/validation.ts`

## 修正内容
006 の実装時に、チェーン検証に渡す配列のソートを `validateAllDefeats` 内で統一的に行い、`validateSlotChain` には「降順ソート済み」の配列しか受け取らないようにする。

関数シグネチャにコメントで降順前提を明記する:

```ts
/**
 * 枠内の湧き-撃破チェーンの整合性を検証する
 * @param sortedSpawns 降順ソート済み（過去→未来）
 * @param sortedDefeats 降順ソート済み（過去→未来）
 */
function validateSlotChain(
  sortedSpawns: readonly SpawnPoint[],
  sortedDefeats: readonly DefeatPoint[],
): boolean {
```

## 完了条件
- ソートが validateAllDefeats 内で統一的に行われていること
- validateSlotChain のJSDocに降順前提が明記されていること
- 006 と同時に対応可能

## 備考
006 の実装レビュー時に確認する項目として扱う。
