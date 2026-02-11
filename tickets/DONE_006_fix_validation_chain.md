# 006: isConsistentDefeat が撃破時刻と湧き時刻の同時（等号）を正しく扱えていない

## 種別
バグ（致命的）

## 症状
湧いていないタイミング（湧き点が存在しないフレーム）に撃破マーカーを追加・配置できてしまう。

## 原因分析

`isConsistentDefeat` の条件 `lastSpawn.frameTime >= defeat.frameTime` は、設計書 §8.5 に「撃破時刻以上（同時含む過去側）」と定義されている。これ自体は正しい。

しかし、**撃破時刻と湧き時刻が完全に等しいケース**（`lastSpawn.frameTime === defeat.frameTime`）が valid と判定される。

ゲームの実態: 湧いた瞬間（0フレーム後）に撃破することはタイムライン操作上はありえないが、バリデーション上は許容される。これ自体は設計書 §8.5 の「同時含む」に合致するので問題ない。

**真の問題は別にある**: `isConsistentDefeat` が「自分自身の撃破から生成された湧き点」を `lastSpawn` として参照してしまうケース。

### 具体例

初期状態: A枠の湧きは 6000F（自動湧き）のみ。

ユーザーが 4000F に撃破を追加しようとする:
1. `testDefeats = [{ frameTime: 4000, slot: 'A' }]`
2. `testSpawns = calculateSpawns(...)`:
   - auto-a: 6000F
   - spawn from defeat: `4000 - 214 = 3786F`
3. `isConsistentDefeat(defeat=4000F, spawns=[6000F, 3786F])`:
   - pastSpawns (>= 4000): [6000F] → lastSpawn = 6000F ✓
   - futureSpawns (< 4000): [3786F] → nextSpawn = 3786F
   - `4000 > 3786` → true → **VALID** ✓

ここまでは正しい。6000F で湧いて 4000F で撃破、次の湧きは 3786F。

**問題ケース: 上記の撃破(4000F)がある状態でさらに 3000F に追加**:
1. `testDefeats = [{ frameTime: 4000 }, { frameTime: 3000 }]`
2. `testSpawns`:
   - auto-a: 6000F
   - from 4000F defeat: 3786F
   - from 3000F defeat: 2786F
3. `isConsistentDefeat(defeat=3000F, spawns=[6000F, 3786F, 2786F])`:
   - pastSpawns (>= 3000): [6000F, 3786F] → lastSpawn = 3786F ✓
   - futureSpawns (< 3000): [2786F] → nextSpawn = 2786F
   - `3000 > 2786` → true → **VALID** ✓

これも正しい。3786F で湧いて 3000F で撃破。

**真に問題のあるケース: 湧き(3786F)の前に撃破、例えば 3800F をさらに追加**:
1. `testDefeats = [{ frameTime: 4000 }, { frameTime: 3800 }]`
2. `testSpawns`:
   - auto-a: 6000F
   - from 4000F defeat: 3786F
   - from 3800F defeat: 3586F
3. `isConsistentDefeat(defeat=3800F, spawns=[6000F, 3786F, 3586F])`:
   - pastSpawns (>= 3800): [6000F] → lastSpawn = 6000F
   - futureSpawns (< 3800): [3786F, 3586F] → nextSpawn = 3786F
   - `3800 > 3786` → true → **VALID** ✓

**これは間違い！** 6000F で湧いて 4000F で撃破されたのに、3800F でまだ存在しているかのように扱われている。3800F の時点ではこの枠のグリルは既に 4000F で撃破されており、次の湧き(3786F)がまだ来ていない。

## 根本原因

`isConsistentDefeat` は「その撃破より過去に湧き点があるか」だけを見ており、**「その湧きが他の撃破によって消費済みかどうか」を考慮していない**。

設計書 §8.5 の疑似コードも同じ問題を持っている。これは**設計上の不備**。

## 正しいバリデーションロジック

枠ごとに「湧き→撃破→湧き→撃破→…」の時系列チェーンが成立するかを検証する必要がある:

```
validateSlotChain(slotSpawns, slotDefeats):
  // 降順ソート（過去→未来）
  sortedSpawns = sort(slotSpawns, desc by frameTime)
  sortedDefeats = sort(slotDefeats, desc by frameTime)

  spawnIdx = 0
  defeatIdx = 0

  while defeatIdx < sortedDefeats.length:
    defeat = sortedDefeats[defeatIdx]

    // この撃破に対応する湧き（撃破以上で最も近い湧き）を探す
    while spawnIdx < sortedSpawns.length && sortedSpawns[spawnIdx].frameTime < defeat.frameTime:
      spawnIdx++

    // 見つけた湧きが撃破以前（>=）でなければ不正
    if spawnIdx >= sortedSpawns.length:
      return false  // 対応する湧きがない
    
    spawn = sortedSpawns[spawnIdx]
    if spawn.frameTime < defeat.frameTime:
      return false  // 湧きが撃破より未来 → 不正

    // この湧きはこの撃破で消費された → 次の湧きへ進む
    spawnIdx++
    defeatIdx++

  return true
```

重要なのは、**1つの湧きは1つの撃破にしか対応できない**ということ。湧き→撃破の1対1マッチングを降順（時間的に過去から）に行う。

## 修正箇所
- `docs/02_GAME_MECHANICS.md` — §8.5 の isConsistentDefeat を修正
- `docs/05_ARCHITECTURE.md` — §4.3 の疑似コードを修正
- `src/utils/validation.ts` — `validateAllDefeats` と `isConsistentDefeat` を置換

## 修正内容

`isConsistentDefeat` を個別チェックから、枠単位のチェーン検証に変更する:

```ts
function validateAllDefeats(
  defeats: readonly DefeatPoint[],
  hazardConfig: InterpolatedHazardConfig,
  directions: readonly DirectionSetting[],
): ValidationResult {
  const testSpawns = calculateSpawns(hazardConfig, directions, defeats);

  // 枠ごとにチェーン検証
  for (const slot of ['A', 'B'] as const) {
    const slotSpawns = testSpawns
      .filter((s) => s.slot === slot)
      .sort((a, b) => b.frameTime - a.frameTime); // 降順
    const slotDefeats = defeats
      .filter((d) => d.slot === slot)
      .sort((a, b) => b.frameTime - a.frameTime); // 降順

    if (!validateSlotChain(slotSpawns, slotDefeats)) {
      return {
        valid: false,
        reason: `${slot}枠の湧き-撃破チェーンが不整合`,
      };
    }
  }

  return { valid: true };
}

function validateSlotChain(
  sortedSpawns: readonly SpawnPoint[],
  sortedDefeats: readonly DefeatPoint[],
): boolean {
  let spawnIdx = 0;

  for (const defeat of sortedDefeats) {
    // この撃破に対応する湧きを探す（defeat.frameTime 以上の湧き）
    while (
      spawnIdx < sortedSpawns.length &&
      sortedSpawns[spawnIdx]!.frameTime < defeat.frameTime
    ) {
      spawnIdx++;
    }

    if (spawnIdx >= sortedSpawns.length) {
      return false; // 対応する湧きがない
    }

    // この湧きはこの撃破で消費 → 次へ
    spawnIdx++;
  }

  return true;
}
```

注意: `isConsistentDefeat` は `findCascadeRemovals` からも呼ばれているため、カスケード判定も同様にチェーン検証方式に変更する必要がある。

`findCascadeRemovals` の修正:

```ts
export function findCascadeRemovals(
  removedId: string,
  allDefeats: readonly DefeatPoint[],
  hazardConfig: InterpolatedHazardConfig,
  directions: readonly DirectionSetting[],
): readonly string[] {
  const removedIds: string[] = [removedId];
  let remaining = allDefeats.filter((d) => d.id !== removedId);
  let changed = true;

  while (changed) {
    changed = false;
    const spawns = calculateSpawns(hazardConfig, directions, remaining);

    for (const slot of ['A', 'B'] as const) {
      const slotSpawns = spawns
        .filter((s) => s.slot === slot)
        .sort((a, b) => b.frameTime - a.frameTime);
      const slotDefeats = remaining
        .filter((d) => d.slot === slot)
        .sort((a, b) => b.frameTime - a.frameTime);

      // チェーンで不正な撃破を特定
      const invalidIds = findInvalidDefeatsInChain(slotSpawns, slotDefeats);
      if (invalidIds.length > 0) {
        remaining = remaining.filter((d) => !invalidIds.includes(d.id));
        removedIds.push(...invalidIds);
        changed = true;
      }
    }
  }

  return removedIds;
}

function findInvalidDefeatsInChain(
  sortedSpawns: readonly SpawnPoint[],
  sortedDefeats: readonly DefeatPoint[],
): string[] {
  const invalidIds: string[] = [];
  let spawnIdx = 0;

  for (const defeat of sortedDefeats) {
    while (
      spawnIdx < sortedSpawns.length &&
      sortedSpawns[spawnIdx]!.frameTime < defeat.frameTime
    ) {
      spawnIdx++;
    }

    if (spawnIdx >= sortedSpawns.length) {
      // 残りの撃破はすべて不正
      invalidIds.push(defeat.id);
      continue;
    }

    // 消費して次へ
    spawnIdx++;
  }

  return invalidIds;
}
```

## 設計書の更新

`docs/02_GAME_MECHANICS.md` の §8.5 を以下に書き換える:

> isConsistentDefeat の個別チェック方式では、1つの湧きが複数の撃破に対応してしまう問題がある。正しくは枠単位で「湧き→撃破→湧き→撃破→…」のチェーンが1対1で成立するかを検証する（validateSlotChain）。

## 完了条件
- 撃破マーカーを、対応する湧きが存在しないタイミング（前の湧きが他の撃破で消費済み）に追加できないこと
- 既存の正当な撃破（6000F→撃破→湧き→撃破→…の連鎖）は従来通り追加できること
- `docs/02_GAME_MECHANICS.md` §8.5 が更新されていること
- テストケースに §8.4 の例（3800F問題）を追加して通ること
- Playwright で手動確認: 湧きのないタイミングにクリックしても撃破が追加されないこと
