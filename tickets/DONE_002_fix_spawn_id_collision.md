# 002: SpawnPoint.id が DefeatPoint.id と衝突するリスク

## 種別
バグ（潜在的ID衝突）

## 問題
`src/utils/calculations.ts` の `calculateSpawns` で、撃破由来の湧き点を生成する際:

```ts
result.push({
  id: defeat.id,       // ← DefeatPoint.id をそのまま使用
  ...
  defeatId: defeat.id,
});
```

SpawnPoint.id と DefeatPoint.id が同じ値になるため、spawns と defeats を混合するリストや Map で使った場合に衝突する。React の key としても同一スロット内で spawn と defeat が同じ key を持つ可能性がある。

## 修正箇所
- `src/utils/calculations.ts` — `calculateSpawns` 関数内の撃破由来 SpawnPoint 生成部分

## 修正内容
SpawnPoint.id にプレフィックスを付けて一意性を確保する:

```ts
result.push({
  id: `spawn-${defeat.id}`,
  slot: defeat.slot,
  frameTime: spawnTime,
  direction: getDirectionAtTime(spawnerDecisionTime, sortedDirections),
  isAuto: false,
  defeatId: defeat.id,
});
```

## 完了条件
- 撃破由来の SpawnPoint.id が DefeatPoint.id と異なる値になっていること
- `defeatId` フィールドは従来通り defeat.id を保持すること
- 既存テストが通ること
