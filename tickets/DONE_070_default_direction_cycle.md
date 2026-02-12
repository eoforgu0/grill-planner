# 070: 方面設定のデフォルトを0→1→2→0...の循環にする

## 種別
UI改善

## 問題
現在 `generateDefaultDirections` は全区間を `direction: 1`（正面）にしている。デフォルトは1つ目から順に 0→1→2→0→1→2... と循環させたい。

## 修正箇所
- `src/utils/calculations.ts` — `generateDefaultDirections`

## 修正内容

```ts
// 変更前
export function generateDefaultDirections(directionInterval: number): readonly DirectionSetting[] {
  const times = getDirectionSwitchTimes(directionInterval);
  return times.map((frameTime) => ({
    frameTime,
    direction: 1 as DirectionId,
  }));
}

// 変更後
export function generateDefaultDirections(directionInterval: number): readonly DirectionSetting[] {
  const times = getDirectionSwitchTimes(directionInterval);
  return times.map((frameTime, i) => ({
    frameTime,
    direction: (i % 3) as DirectionId,  // 0, 1, 2, 0, 1, 2, ...
  }));
}
```

これにより:
- 1つ目（100秒）: 0（プリセットの1番目 = デフォルトで「左」）
- 2つ目: 1（プリセットの2番目 = デフォルトで「正面」）
- 3つ目: 2（プリセットの3番目 = デフォルトで「右」）
- 4つ目: 0（「左」）
- ...

## 完了条件
- 初期状態およびキケン度変更で方面数が変わった際のデフォルトが 0→1→2→0... の循環であること
- プリセット名がデフォルト（「左」「正面」「右」）の場合、タイムライン上で左→正面→右→左...と表示されること
