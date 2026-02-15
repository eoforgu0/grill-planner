# 092: 湧きフキダシに方面内通し番号を表示

## 種別
UI改善

## 概要

湧きマーカーのフキダシに、その方面で何体目の湧きかを示す通し番号をカッコ内に表示する。

表示例: `100.0s 正面(1) 1P`、`92.4s 左(2) 3P`

## 修正箇所

### SpawnMarker.tsx — SpawnDisplayInfo に通し番号フィールド追加

```ts
export interface SpawnDisplayInfo {
  directionName: string;
  directionIndex: number;  // 追加: 方面内の通し番号（1始まり）
  targetLabel: string | null;
  targetIcon: string | null;
}
```

フキダシ表示:

現在:
```tsx
{seconds}s {dirName}
{targetLabel && displayMode !== "icon" && <> {targetLabel}</>}
```

変更後:
```tsx
{seconds}s {dirName}({displayInfo?.directionIndex ?? "?"})
{targetLabel && displayMode !== "icon" && <> {targetLabel}</>}
```

### Timeline/index.tsx — spawnDisplayMap 構築時に方面別カウンター

```tsx
const spawnDisplayMap: ReadonlyMap<string, SpawnDisplayInfo> = useMemo(() => {
  const allSpawns = [...spawns].filter((s) => s.frameTime > 0).sort((a, b) => b.frameTime - a.frameTime);

  // 方面別カウンター（DirectionId → 現在のカウント）
  const directionCounters = new Map<number, number>();

  const map = new Map<string, SpawnDisplayInfo>();
  for (let i = 0; i < allSpawns.length; i++) {
    const spawn = allSpawns[i]!;
    const directionName = directionPresets[spawn.direction] ?? `方面${spawn.direction + 1}`;

    // 方面内通し番号（1始まり）
    const currentCount = (directionCounters.get(spawn.direction) ?? 0) + 1;
    directionCounters.set(spawn.direction, currentCount);

    let targetLabel: string | null = null;
    let targetIcon: string | null = null;
    const targetEntry = targetOrder[i];
    if (targetEntry && targetEntry !== "-") {
      targetLabel = targetEntry;
      const playerIndex = Number.parseInt(targetEntry[0]!, 10) - 1;
      const weaponRowId = weapons[playerIndex];
      if (weaponRowId) {
        const weapon = weaponMaster.find((w) => w.rowId === weaponRowId);
        targetIcon = weapon ? getWeaponIconPath(weapon.id) : null;
      }
    }

    map.set(spawn.id, { directionName, directionIndex: currentCount, targetLabel, targetIcon });
  }
  return map;
}, [spawns, directionPresets, targetOrder, weapons, weaponMaster]);
```

`allSpawns` は frameTime 降順（過去→未来）にソートされているため、カウンターは時間の早い湧き（過去）から順に 1, 2, 3... と振られる。

## 完了条件
- 湧きフキダシに方面名の直後にカッコ付き通し番号が表示されること（例: `正面(1)`）
- 通し番号は方面ごとに独立してカウントされること
- 通し番号は時間順（過去→未来、frameTime 降順）で1から振られること
- A枠・B枠をまたいで同じ方面であれば通しでカウントされること
