# 040: 方面帯の色を同一方面名で同一カラーにする

## 種別
UI改善

## 問題
現在は方面帯の色が区間インデックスで決まる（`DIR_BAND_COLORS[index % 9]`）ため、同じ方面名（例: 区間1「左」と区間4「左」）でも異なる色になる。直感的でない。

## 修正箇所
- `src/components/Timeline/DirectionBands.tsx` — 色決定ロジック
- `src/components/Timeline/DirectionLabels.tsx` — 同様に方面ラベル背景色の決定ロジック

## 修正内容
方面名からカラーインデックスを決定する。同じ方面名には同じ色を割り当てる。

### 方針
方面設定の中からユニークな方面名を抽出し、出現順にカラーインデックスを割り当てる:

```ts
function getDirectionColorMap(
  directions: readonly DirectionSetting[],
): Map<string, string> {
  const sorted = [...directions].sort((a, b) => b.frameTime - a.frameTime);
  const colorMap = new Map<string, string>();
  let colorIdx = 0;

  for (const dir of sorted) {
    if (!colorMap.has(dir.direction)) {
      colorMap.set(
        dir.direction,
        DIR_BAND_COLORS[colorIdx % DIR_BAND_COLORS.length] ?? DIR_BAND_COLORS[0]!,
      );
      colorIdx++;
    }
  }

  return colorMap;
}
```

この関数を DirectionBands と DirectionLabels の両方で使用する。共通ユーティリティとして coordinates.ts に配置するか、両コンポーネントに渡す方式でもよい。

### 使用側

```tsx
// DirectionBands.tsx
const colorMap = getDirectionColorMap(directions);
// ...
const color = colorMap.get(dir.direction) ?? DIR_BAND_COLORS[0];
```

## 完了条件
- 同じ方面名の区間が同じ色で表示されること（例: 「左」が複数区間にある場合、すべて同色）
- 異なる方面名は異なる色であること（9色以内であれば）
- 方面帯（DirectionBands）と方面ラベル（DirectionLabels）で色が一致すること
