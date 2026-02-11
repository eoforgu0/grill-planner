# 004: DirectionLabels の originalIndex 検索を安全にする

## 種別
バグ（方面名更新先のインデックスずれリスク）

## 問題
`src/components/Timeline/DirectionLabels.tsx` で、ソート後の方面リストから元の directions 配列のインデックスを `findIndex` で逆引きしている:

```ts
const originalIndex = directions.findIndex(
  (d) => d.frameTime === dir.frameTime && d.direction === dir.direction,
);
```

この方法は `frameTime` と `direction` の組み合わせが重複した場合に誤ったインデックスを返す可能性がある（同名の方面が複数ある場合など）。

また、directions が常に降順で保持されているか明示的に保証されていないため、ソート前後でインデックスがずれると `UPDATE_DIRECTION_NAME` が間違った区間を更新する。

## 修正箇所
- `src/components/Timeline/DirectionLabels.tsx`

## 修正内容
ソート前の directions 配列のインデックスをそのまま使う形に変更する。

具体的には、ソート時に元のインデックスを保持する:

```tsx
const sortedDirs = directions
  .map((dir, originalIndex) => ({ ...dir, originalIndex }))
  .sort((a, b) => b.frameTime - a.frameTime);
```

そして各ラベルで `sortedDirs[i].originalIndex` を使用する。これにより findIndex による逆引きが不要になる。

## 完了条件
- `findIndex` による逆引きが削除されていること
- ソート時に元のインデックスが保持されていること
- 方面名の編集が正しいインデックスに反映されること（手動確認）
