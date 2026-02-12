# 047: 方面帯の色を内部ID(0,1,2)ごとに固定色にする

## 種別
UI改善

## 問題
現在は `getDirectionColorMap` で出現順に `DIR_BAND_COLORS` を割り当てているが、方面IDが固定の3値（0, 1, 2）なので、各IDに固定の色を割り当てた方がわかりやすい。

## 修正箇所
- `src/index.css` — 方面色の CSS カスタムプロパティを変更
- `src/components/Timeline/coordinates.ts` — `getDirectionColorMap` を固定マッピングに変更

## 修正内容

### CSS カスタムプロパティ（既存の dir-1〜9 を置き換え）

```css
/* 方面ID固定色 */
--color-dir-0: #fef9c3;  /* ID 0 — 薄い黄 (yellow-100) */
--color-dir-1: #fee2e2;  /* ID 1 — 薄い赤 (red-100) */
--color-dir-2: #dbeafe;  /* ID 2 — 薄い青 (blue-100) */
```

dir-3〜9 は不要になるので削除してよい。

### 固定マッピング

```ts
export const DIRECTION_ID_COLORS: Record<DirectionId, string> = {
  0: 'var(--color-dir-0)',
  1: 'var(--color-dir-1)',
  2: 'var(--color-dir-2)',
} as const;

export function getDirectionColor(id: DirectionId): string {
  return DIRECTION_ID_COLORS[id];
}
```

`getDirectionColorMap` は廃止し、使用箇所を `getDirectionColor(dir.direction)` に置き換える。

### 使用箇所の修正
- `DirectionBands.tsx` — `getDirectionColor(dir.direction)`
- `DirectionLabels.tsx` — 同上

## 完了条件
- 方面ID 0 が常に薄い黄色で表示されること
- 方面ID 1 が常に薄い赤色で表示されること
- 方面ID 2 が常に薄い青色で表示されること
- 同じIDの複数区間が同色であること
