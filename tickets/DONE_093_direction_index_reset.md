# 093: 方面内通し番号を方面切替ごとにリセットする

## 種別
バグ修正（仕様認識ズレ）

## 概要

チケット092で追加した方面内通し番号は、方面ごとに全体を通してカウントしていた。
正しくは、方面が切り替わるたびにカウンターをリセットし、同じ方面の連続湧きの中での何体目かを示す。

## 現状の挙動（092実装）

「右→右→左→左→右→右」の湧き順の場合:
```
右(1), 右(2), 左(1), 左(2), 右(3), 右(4)
```

## 期待する挙動

```
右(1), 右(2), 左(1), 左(2), 右(1), 右(2)
```

方面が切り替わった時点でカウンターが1に戻る。

## 修正箇所

### Timeline/index.tsx — spawnDisplayMap 構築ロジック

方面別カウンターを「前回の方面」との比較でリセットする方式に変更:

```tsx
const spawnDisplayMap: ReadonlyMap<string, SpawnDisplayInfo> = useMemo(() => {
  const allSpawns = [...spawns].filter((s) => s.frameTime > 0).sort((a, b) => b.frameTime - a.frameTime);

  let prevDirection: number | null = null;
  let directionCounter = 0;

  const map = new Map<string, SpawnDisplayInfo>();
  for (let i = 0; i < allSpawns.length; i++) {
    const spawn = allSpawns[i]!;
    const directionName = directionPresets[spawn.direction] ?? `方面${spawn.direction + 1}`;

    // 方面が前回と異なればカウンターリセット
    if (spawn.direction !== prevDirection) {
      directionCounter = 1;
      prevDirection = spawn.direction;
    } else {
      directionCounter++;
    }

    // ... targetLabel, targetIcon の処理は変更なし ...

    map.set(spawn.id, { directionName, directionIndex: directionCounter, targetLabel, targetIcon });
  }
  return map;
}, [spawns, directionPresets, targetOrder, weapons, weaponMaster]);
```

`Map<DirectionId, number>` のカウンターは不要になるため削除。
代わりに `prevDirection` と `directionCounter` のスカラー変数で管理。

## 注意

`allSpawns` は frameTime 降順（過去→未来）でソートされているため、ゲーム内の時間進行順に走査している。
方面切り替えの検知はこの順序に依存する。

## 完了条件
- 「右→右→左→左→右→右」の場合に `右(1), 右(2), 左(1), 左(2), 右(1), 右(2)` と表示されること
- 同一方面が途切れなく続く場合は連続カウント（例: 右が5連続なら `右(1)〜右(5)`）
- 間に別方面が1体でも挟まればカウンターがリセットされること
