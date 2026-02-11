# 043: 方面プリセット名変更時に既存の方面選択も連動変更する（内部IDベースに変更）

## 種別
バグ（仕様不一致）

## 問題
方面名は内部的には3つのスロットがあり、プリセット名はその表示ラベルにすぎない。プリセット名を「正面」→「裏」に変更した場合、既にタイムライン上で「正面」と設定されている方面区間も「裏」に変わるべきだが、現在はプリセット名の変更がラベル変更のみで、既存の方面選択箇所に反映されない。

## 設計方針: 内部IDベース

方面の選択値を表示名の文字列ではなく、固定の内部ID（`0 | 1 | 2`）で管理する。表示時にプリセット配列 `directionPresets[id]` で表示名を引く。

- 内部IDは `directionPresets` 配列のインデックスと直接対応するため、変換が不要
- プリセット名を変更しても内部IDは変わらないため、既存の方面選択が自動的に新しい名前で表示される
- 文字列比較による一致判定の壊れやすさを回避
- エクスポート時は内部IDを保存、表示名はプリセットから復元

## 修正箇所
- `src/types/base.ts` — DirectionId 型の追加
- `src/types/game.ts` — DirectionSetting.direction を DirectionId に変更
- `src/utils/calculations.ts` — generateDefaultDirections のデフォルト値変更
- `src/components/Timeline/DirectionLabels.tsx` — 内部ID→表示名の変換
- `src/components/Timeline/DirectionBands.tsx` — 同上
- `src/components/Timeline/coordinates.ts` — getDirectionColorMap の引数変更
- `src/utils/statistics.ts` — 統計の方面名を表示名に変換
- `src/components/Statistics/DirectionStatsTable.tsx` — 必要に応じて
- `src/hooks/scenarioReducer.ts` — SET_DIRECTION_PRESET の簡素化
- `src/utils/fileIO.ts` — エクスポート/インポート互換性

## 修正内容

### 1. 型の追加

```ts
// src/types/base.ts
export type DirectionId = 0 | 1 | 2;
```

### 2. DirectionSetting の変更

```ts
// src/types/game.ts
export interface DirectionSetting {
  readonly frameTime: FrameTime;
  readonly direction: DirectionId;  // 表示名ではなく内部ID
}
```

### 3. 表示名の解決

```ts
// src/utils/directions.ts（新規、または calculations.ts に追加）
export function resolveDirectionName(
  id: DirectionId,
  presets: readonly [string, string, string],
): string {
  return presets[id] ?? `方面${id + 1}`;
}
```

### 4. generateDefaultDirections の変更

```ts
// 変更前
return times.map((frameTime) => ({
  frameTime,
  direction: '正面',
}));

// 変更後: デフォルトは内部ID 1（プリセットの2番目 = デフォルトで「正面」）
return times.map((frameTime) => ({
  frameTime,
  direction: 1 as DirectionId,
}));
```

### 5. 色マッピングの変更

`getDirectionColorMap` を表示名ベースから内部IDベースに変更:

```ts
export function getDirectionColorMap(
  directions: readonly DirectionSetting[],
): Map<DirectionId, string> {
  const colorMap = new Map<DirectionId, string>();
  const sorted = [...directions].sort((a, b) => b.frameTime - a.frameTime);
  let colorIdx = 0;

  for (const dir of sorted) {
    if (!colorMap.has(dir.direction)) {
      colorMap.set(dir.direction, DIR_BAND_COLORS[colorIdx % DIR_BAND_COLORS.length]!);
      colorIdx++;
    }
  }
  return colorMap;
}
```

### 6. 表示系の修正

DirectionLabels, DirectionBands, statistics 等で `direction`（内部ID）を表示する箇所に `resolveDirectionName(dir.direction, presets)` を適用する。presets は props で上位から渡す。

### 7. DirectionLabels のプリセット選択

フロートのボタンクリック時に表示名ではなく内部IDを渡す:

```tsx
{presetNames.map((preset, presetId) => (
  <button
    key={presetId}
    onClick={() => selectPreset(presetId as DirectionId)}
    ...
  >
    {preset}
  </button>
))}
```

### 8. SET_DIRECTION_PRESET の簡素化

プリセット名変更時に directions の更新は不要（内部IDは変わらないため、表示が自動的に新しい名前になる）。既存の実装のままでよい。

### 9. エクスポート/インポート互換性

エクスポート時: direction フィールドに内部ID（数値）を保存。

インポート時: 旧形式（文字列の表示名）が含まれている場合に変換:

```ts
function migrateDirectionId(
  direction: string | number,
  presets: readonly [string, string, string],
): DirectionId {
  if (typeof direction === 'number' && [0, 1, 2].includes(direction)) {
    return direction as DirectionId;
  }
  if (typeof direction === 'string') {
    const idx = presets.indexOf(direction);
    if (idx >= 0 && idx <= 2) return idx as DirectionId;
  }
  return 1; // フォールバック: 「正面」相当
}
```

## 完了条件
- 方面プリセット名を変更すると、タイムラインの全方面ラベルが即座に新しい名前で表示されること
- 方面帯の色（同ID=同色）が正しく動作すること
- 統計テーブルの方面名が正しく表示されること
- 方面選択フロートのボタンラベルがプリセット名に連動すること
- エクスポート/インポートが正しく動作すること（旧形式との互換性含む）
