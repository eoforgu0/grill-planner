# 074: グローバル湧き抑制（Spawn Suppression）の実装

## 種別
ゲーム仕様追加（計算ロジック変更）

## 背景

グリルにはグローバルな湧き抑制メカニズムがある:
**「グリルが湧いた後184F（3.07秒）間は、A枠・B枠問わず次のグリルが湧けない」**

これは「必ず184F後に湧く」というロジックではなく、「184F間は湧けない制約があり、本来の湧きフレームが抑制期間内に入る場合のみ遅延する」という仕様。

### 実測データによる検証

**ケース1:** A枠B枠同時撃破 @ 残り62.23秒 (3734F)
- 3匹目: 58.67s (3520F) = 3734 - 214 → 通常通り
- 4匹目: 55.62s (3337F) ≈ 3520 - 184 = 3336F → 抑制適用（3520Fに湧けないため抑制明けまで遅延）

**ケース2:** 1匹撃破 @ 残り90.72秒 (5443F)
- 2匹目: 87.15s (5229F) = 5443 - 214 → 撃破由来の通常リスポーン
- 3匹目: 84.08s (5045F) = 5229 - 184 → B枠自動湧き（rawFrameTimeは5229F以内）が抑制を受けて遅延

### 抑制が発生するケースの理論的整理

**同一枠の連続撃破では原理的に抑制は発生しない。**
理由: 撃破→湧きの間隔は RESPAWN_FRAMES = 214F。同一枠では「湧き→撃破→湧き」の最短サイクルが214F（湧いた瞬間に撃破しても次の湧きは214F後）であり、抑制期間 184F を常に上回る。

**抑制が発生するのは異なる枠の湧きが近接するケースに限られる。** 具体的には:
- A枠の撃破由来の湧きとB枠の撃破由来の湧きが近接する場合
- 一方の枠の撃破由来の湧きと、もう一方の枠の自動湧きが近接する場合

## 修正箇所
- `src/types/game.ts` — SpawnPoint に `isSuppressed` フィールド追加
- `src/constants/index.ts` — `SPAWN_SUPPRESSION_FRAMES` 定数追加
- `src/utils/calculations.ts` — `calculateSpawns` 書き換え
- `src/__tests__/calculations.test.ts` — 抑制テストケース追加
- `src/components/Timeline/SpawnMarker.tsx` — 抑制された湧きの視覚表示

## 修正内容

### 1. 定数追加 (`constants/index.ts`)

```ts
/** グローバル湧き抑制フレーム数: 湧き後184F間は次の湧きが発生できない */
export const SPAWN_SUPPRESSION_FRAMES = 184 as const;
```

SPAWNER_DECISION_FRAMES と同じ値(184)だが、意味が異なる:
- SPAWNER_DECISION_FRAMES: 撃破→スポナー決定の時間差
- SPAWN_SUPPRESSION_FRAMES: 湧き→次に湧けるようになるまでの抑制期間

### 2. 型拡張 (`types/game.ts`)

```ts
export interface SpawnPoint {
  readonly id: string;
  readonly slot: GrillSlot;
  readonly frameTime: FrameTime;
  readonly direction: DirectionId;
  readonly isAuto: boolean;
  readonly defeatId?: string;
  readonly isSuppressed?: boolean;      // 抑制により遅延したか
  readonly rawFrameTime?: FrameTime;    // 抑制前の本来の湧きフレーム（抑制時のみ設定）
}
```

### 3. calculateSpawns 書き換え (`utils/calculations.ts`)

#### アルゴリズム概要

```
Phase 1: 全湧きイベントの「生フレーム（rawFrameTime）」を計算（従来と同じ）
Phase 2: rawFrameTime の降順（ゲーム進行順）にソートし、抑制を適用
```

#### 詳細実装

```ts
interface PendingSpawn {
  id: string;
  slot: GrillSlot;
  rawFrameTime: FrameTime;
  direction: DirectionId;
  isAuto: boolean;
  defeatId?: string;
}

export function calculateSpawns(
  hazardConfig: InterpolatedHazardConfig,
  directions: readonly DirectionSetting[],
  defeats: readonly DefeatPoint[],
): readonly SpawnPoint[] {
  const sortedDirections = [...directions].sort((a, b) => b.frameTime - a.frameTime);

  // === Phase 1: 全湧きの rawFrameTime を計算 ===
  const pending: PendingSpawn[] = [];

  // A枠 自動湧き
  const firstDirection: DirectionId = sortedDirections[0]?.direction ?? 1;
  pending.push({
    id: "auto-a",
    slot: "A",
    rawFrameTime: GAME_DURATION_FRAMES,
    direction: firstDirection,
    isAuto: true,
  });

  // B枠 自動湧き
  if (hazardConfig.bSlotOpenFrame >= 0) {
    const bSlotSpawnerDecision = hazardConfig.bSlotOpenFrame + SPAWN_WAIT_FRAMES;
    pending.push({
      id: "auto-b",
      slot: "B",
      rawFrameTime: hazardConfig.bSlotOpenFrame,
      direction: getDirectionAtTime(bSlotSpawnerDecision, sortedDirections),
      isAuto: true,
    });
  }

  // 撃破由来の湧き
  for (const defeat of defeats) {
    const spawnerDecisionTime = calculateSpawnerDecisionTime(defeat.frameTime);
    const rawSpawnTime = calculateSpawnTime(defeat.frameTime);
    pending.push({
      id: `spawn-${defeat.id}`,
      slot: defeat.slot,
      rawFrameTime: rawSpawnTime,
      direction: getDirectionAtTime(spawnerDecisionTime, sortedDirections),
      isAuto: false,
      defeatId: defeat.id,
    });
  }

  // === Phase 2: 降順ソート → 抑制適用 ===
  // ソート: rawFrameTime 降順（ゲーム進行順）
  // 同フレームの場合: A枠優先
  pending.sort((a, b) => {
    if (b.rawFrameTime !== a.rawFrameTime) return b.rawFrameTime - a.rawFrameTime;
    if (a.slot !== b.slot) return a.slot === "A" ? -1 : 1;
    return 0;
  });

  const result: SpawnPoint[] = [];
  let lastSpawnFrame = Infinity; // まだ何も湧いていない状態

  for (const p of pending) {
    // 抑制チェック: 直前の湧きから184F以内は湧けない
    const suppressionLimit = lastSpawnFrame - SPAWN_SUPPRESSION_FRAMES;
    // フレームは大きい=過去、小さい=未来
    // rawFrameTime > suppressionLimit → まだ抑制期間内 → suppressionLimit まで遅延
    // rawFrameTime <= suppressionLimit → 抑制期間外 → そのまま
    // rawFrameTime == suppressionLimit → ちょうど184F後 → 湧ける（抑制なし）
    const isSuppressed = p.rawFrameTime > suppressionLimit;
    const actualFrameTime = isSuppressed
      ? suppressionLimit
      : p.rawFrameTime;

    result.push({
      id: p.id,
      slot: p.slot,
      frameTime: actualFrameTime,
      direction: p.direction,
      isAuto: p.isAuto,
      defeatId: p.defeatId,
      ...(isSuppressed ? { isSuppressed: true, rawFrameTime: p.rawFrameTime } : {}),
    });

    lastSpawnFrame = actualFrameTime;
  }

  return result;
}
```

#### 初期値 `lastSpawnFrame = Infinity` の意味

- Infinity - 184 = Infinity なので、最初の湧き（通常は6000FのA枠自動湧き）は抑制されない
- これは「ゲーム開始前には何も湧いていない」を表現

#### 境界値 `rawFrameTime == suppressionLimit` の扱い

ちょうど184F後（`rawFrameTime == suppressionLimit`）は **抑制されない（湧ける）**。
判定は `>` であり `>=` ではない。「184F間は湧けない」＝「184F経過した後は湧ける」。

### 4. テストケース (`__tests__/calculations.test.ts`)

```ts
describe("calculateSpawns — spawn suppression", () => {
  const hazardConfig: InterpolatedHazardConfig = {
    dozerIncrSecond: 30,
    waveChangeNum: 5,
    directionInterval: 14.4,
    bSlotOpenFrame: 3786,
  };
  const directions = generateDefaultDirections(14.4);

  it("通常: 抑制が発生しない場合はrawFrameTimeと同じ", () => {
    // A枠撃破 @ 5400F → rawSpawn = 5186F
    // lastSpawnFrame = 6000 (auto-a)
    // suppressionLimit = 6000 - 184 = 5816
    // 5186 < 5816 → 抑制なし
    const defeats: DefeatPoint[] = [{ id: "d1", slot: "A", frameTime: 5400 }];
    const spawns = calculateSpawns(hazardConfig, directions, defeats);
    const d1Spawn = spawns.find(s => s.defeatId === "d1");
    expect(d1Spawn?.frameTime).toBe(5186);
    expect(d1Spawn?.isSuppressed).toBeUndefined();
  });

  it("抑制: A枠B枠の同時撃破で後発の湧きが遅延する（ケース1再現）", () => {
    // pending降順: auto-a(6000), auto-b(3786), d1(A,3520), d2(B,3520)
    //
    // auto-a: 6000, last=∞ → ok, last=6000
    // auto-b: 3786, limit=5816, 3786<5816 → ok, last=3786
    // d1(A):  3520, limit=3602, 3520<3602 → ok, last=3520
    // d2(B):  3520, limit=3336, 3520>3336 → 抑制! actual=3336
    const defeats: DefeatPoint[] = [
      { id: "d1", slot: "A", frameTime: 3734 },
      { id: "d2", slot: "B", frameTime: 3734 },
    ];
    const spawns = calculateSpawns(hazardConfig, directions, defeats);

    const d1Spawn = spawns.find(s => s.defeatId === "d1");
    const d2Spawn = spawns.find(s => s.defeatId === "d2");

    // A枠（優先）: 抑制なし
    expect(d1Spawn?.frameTime).toBe(3520);
    expect(d1Spawn?.isSuppressed).toBeUndefined();

    // B枠: 抑制あり、3520 - 184 = 3336F
    expect(d2Spawn?.frameTime).toBe(3336);
    expect(d2Spawn?.isSuppressed).toBe(true);
    expect(d2Spawn?.rawFrameTime).toBe(3520);
  });

  it("抑制: B枠自動湧きがA枠撃破由来の湧きに抑制される（ケース2類似）", () => {
    // B枠自動湧きの rawFrameTime が、A枠の撃破由来の湧きの 184F 以内にある
    // ケースを再現する。
    //
    // bSlotOpenFrame = 3786F のconfig を使用。
    // A枠撃破を、A枠の湧きが3786Fの184F以内に入るタイミングに設定する。
    //
    // rawSpawn = defeatFrame - 214 が 3786+184 = 3970 以内であればよい。
    // つまり defeatFrame - 214 > 3786 - 184 = 3602 かつ rawSpawn < 3786 あたり。
    //
    // ここでは rawSpawn を 3900F にしたい → defeatFrame = 3900+214 = 4114F
    //
    // pending降順: auto-a(6000), d1(A,3900), auto-b(B,3786)
    // auto-a: 6000, last=∞ → ok, last=6000
    // d1(A):  3900, limit=5816, 3900<5816 → ok, last=3900
    // auto-b: 3786, limit=3716, 3786>3716 → 抑制! actual=3716
    const defeats: DefeatPoint[] = [
      { id: "d1", slot: "A", frameTime: 4114 },
    ];
    const spawns = calculateSpawns(hazardConfig, directions, defeats);

    const d1Spawn = spawns.find(s => s.defeatId === "d1");
    const autoB = spawns.find(s => s.id === "auto-b");

    // A枠撃破由来: 抑制なし
    expect(d1Spawn?.frameTime).toBe(3900);
    expect(d1Spawn?.isSuppressed).toBeUndefined();

    // B枠自動湧き: 抑制あり、3900 - 184 = 3716F
    expect(autoB?.frameTime).toBe(3716);
    expect(autoB?.isSuppressed).toBe(true);
    expect(autoB?.rawFrameTime).toBe(3786);
  });

  it("境界値: ちょうど184F後の湧きは抑制されない", () => {
    // A枠の湧きが X フレームに確定した後、
    // B枠の rawFrameTime がちょうど X - 184 のケース。
    // → 抑制されない（`>` 判定であり `>=` ではない）
    //
    // auto-a(6000) の184F後 = 5816F に rawSpawn が来るようにする。
    // rawSpawn = 5816 → defeatFrame = 5816 + 214 = 6030F
    // ただし6030FはB枠では撃破不可（ゲーム開始前）なのでA枠で試す。
    //
    // 別アプローチ: 手動でbSlotOpenFrameを調整してauto-bが境界値になるようにする。
    const boundaryConfig: InterpolatedHazardConfig = {
      ...hazardConfig,
      bSlotOpenFrame: 6000 - 184, // = 5816F ちょうどauto-aの184F後
    };
    const spawns = calculateSpawns(boundaryConfig, directions, []);

    const autoA = spawns.find(s => s.id === "auto-a");
    const autoB = spawns.find(s => s.id === "auto-b");

    expect(autoA?.frameTime).toBe(6000);
    expect(autoB?.frameTime).toBe(5816); // 抑制されない
    expect(autoB?.isSuppressed).toBeUndefined();
  });

  it("境界値: 184F未満（183F後）の湧きは抑制される", () => {
    const boundaryConfig: InterpolatedHazardConfig = {
      ...hazardConfig,
      bSlotOpenFrame: 6000 - 183, // = 5817F auto-aの183F後 → 抑制期間内
    };
    const spawns = calculateSpawns(boundaryConfig, directions, []);

    const autoB = spawns.find(s => s.id === "auto-b");

    // 5817 > 6000-184(=5816) → 抑制される
    expect(autoB?.frameTime).toBe(5816); // 5816Fまで遅延
    expect(autoB?.isSuppressed).toBe(true);
    expect(autoB?.rawFrameTime).toBe(5817);
  });

  it("同一枠の連続撃破では抑制が発生しない（214F > 184F）", () => {
    // 同一枠では撃破→湧きが214F間隔であるため、
    // 湧き同士の最短間隔も214F以上となり、184Fの抑制に引っかからない。
    // 湧いた瞬間（0F後）に撃破しても: rawSpawn = spawnFrame - 214 であり
    // spawnFrame - 214 < spawnFrame - 184（= suppressionLimit）が常に成立する。
    const defeats: DefeatPoint[] = [
      { id: "d1", slot: "A", frameTime: 6000 },   // auto-a を即撃破
      { id: "d2", slot: "A", frameTime: 5786 },   // d1の湧き(5786F)を即撃破
      { id: "d3", slot: "A", frameTime: 5572 },   // d2の湧き(5572F)を即撃破
    ];
    const spawns = calculateSpawns(hazardConfig, directions, defeats);

    const d1Spawn = spawns.find(s => s.defeatId === "d1");
    const d2Spawn = spawns.find(s => s.defeatId === "d2");
    const d3Spawn = spawns.find(s => s.defeatId === "d3");

    // 全て抑制なし（214F間隔 > 184F抑制期間）
    expect(d1Spawn?.frameTime).toBe(5786);
    expect(d1Spawn?.isSuppressed).toBeUndefined();
    expect(d2Spawn?.frameTime).toBe(5572);
    expect(d2Spawn?.isSuppressed).toBeUndefined();
    expect(d3Spawn?.frameTime).toBe(5358);
    expect(d3Spawn?.isSuppressed).toBeUndefined();
  });

  it("抑制なし: 異なる枠でも十分間隔が空いている場合", () => {
    const defeats: DefeatPoint[] = [
      { id: "d1", slot: "A", frameTime: 5400 },
      { id: "d2", slot: "B", frameTime: 3700 },
    ];
    const spawns = calculateSpawns(hazardConfig, directions, defeats);

    // pending降順: auto-a(6000), d1(A,5186), d2(B,3486), auto-b(B,3786)
    // ソート: 6000, 5186, 3786, 3486
    // 全て184F以上離れている → 抑制なし
    const d1Spawn = spawns.find(s => s.defeatId === "d1");
    const d2Spawn = spawns.find(s => s.defeatId === "d2");
    expect(d1Spawn?.isSuppressed).toBeUndefined();
    expect(d2Spawn?.isSuppressed).toBeUndefined();
  });

  it("A枠自動湧きとB枠自動湧きの間に抑制が発生しない（十分な間隔）", () => {
    const spawns = calculateSpawns(hazardConfig, directions, []);
    const autoA = spawns.find(s => s.id === "auto-a");
    const autoB = spawns.find(s => s.id === "auto-b");
    expect(autoA?.isSuppressed).toBeUndefined();
    expect(autoB?.isSuppressed).toBeUndefined();
  });
});
```

### 5. 視覚表示 (`SpawnMarker.tsx`)

抑制された湧きマーカーに控えめな視覚効果を追加:

```tsx
// SpawnMarker 内で isSuppressed を参照
const isSuppressed = spawn.isSuppressed === true;

// マーカー円に点線ボーダーを適用（通常は実線）
<div
  className="shrink-0 rounded-full"
  style={{
    width: MARKER_SIZE,
    height: MARKER_SIZE,
    backgroundColor: "var(--color-spawn)",
    border: `2px ${isSuppressed ? "dashed" : "solid"} ${borderColor}`,
  }}
/>
```

抑制されたマーカーは:
- 円のボーダーが **破線（dashed）** になる
- それ以外は通常と同じ

これにより「一応確認できる」程度の控えめな視覚的区別になる。

### 6. バリデーションへの影響（変更不要の根拠）

`validation.ts` は以下のフローで動作する:
1. `validateAllDefeats` → `calculateSpawns` を呼んで湧きを取得
2. 枠ごとに `validateSlotChain` で湧き-撃破の1対1マッチング

`calculateSpawns` が抑制適用済みの `frameTime` を返すようになるため:
- 抑制で湧きが遅延 → `frameTime` が小さくなる → 撃破フレームより小さくなる可能性
- → バリデーションが正しく「対応する湧きがない」と判定する

**ロジック変更不要。**

ただし、テストケースの期待値が変わる可能性があるため、既存テストの確認は必要:
- 現在のテストケースでは抑制が発生するほど近接した撃破は配置していないため、影響なしの見込み

### 7. 注意: 方面判定のタイミング

方面判定は「スポナー決定時刻」で行う。スポナー決定時刻 = `defeatFrame - 184F`。
これは抑制とは無関係（撃破フレームから決まる）なので、方面判定ロジックは変更不要。

抑制で湧きフレームが遅延しても、スポナー決定時刻は変わらない = 方面は変わらない。
（スポナー決定は撃破の184F前に既に行われており、湧きの遅延はその後の話）

### 8. 同一枠で抑制が原理的に発生しない理由の補足

同一枠での撃破→湧きサイクル:
```
湧き(F) → 撃破(F以下) → 次の湧き(撃破F - 214)
```

最短ケース（湧いた瞬間に撃破）:
```
湧き @ X → 撃破 @ X → 次の湧き rawFrame = X - 214
suppressionLimit = X - 184
X - 214 < X - 184 → 常に成立 → 抑制されない
```

つまり同一枠のチェーンでは `rawFrame = 撃破F - 214 <= 前の湧きF - 214 < 前の湧きF - 184 = suppressionLimit` が常に成り立つため、抑制条件 `rawFrame > suppressionLimit` を満たすことはない。

## 完了条件
- `SPAWN_SUPPRESSION_FRAMES = 184` が定数として定義されていること
- `calculateSpawns` が全湧きの rawFrameTime を降順ソートし、184F のグローバル抑制を適用すること
- 同フレームの湧きではA枠が優先されること
- 抑制された SpawnPoint に `isSuppressed: true` と `rawFrameTime` が設定されること
- 抑制されていない SpawnPoint には `isSuppressed` / `rawFrameTime` が設定されないこと
- ちょうど184F後（`rawFrameTime == suppressionLimit`）は抑制されないこと（`>` 判定）
- タイムライン上で抑制された湧きマーカーの円ボーダーが破線になること
- 既存のテストが全て通ること
- 新規テストが全て通ること:
  - A枠B枠同時撃破の抑制（ケース1再現）
  - B枠自動湧きがA枠撃破由来の湧きに抑制されるケース（ケース2類似）
  - 境界値: ちょうど184F後は抑制されない
  - 境界値: 183F後は抑制される
  - 同一枠の連続即撃破は抑制されないことの確認
  - 異なる枠で十分間隔がある場合の非抑制確認
  - A枠B枠自動湧き間の非抑制確認
- バリデーションが抑制後の湧きフレームで正しく判定されること
