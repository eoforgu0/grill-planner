# 059: 湧きマーカーの方面表示を名前に修正し、ターゲット表示を追加し、レーン幅を拡大する

## 種別
バグ + UI改善（機能追加）

## 問題
1. 湧きマーカー右のラベルで方面が内部ID（0, 1, 2）のまま表示されている。プリセット名（「左」「正面」「右」等）で表示すべき。
2. ターゲット順が入力されていれば、湧きマーカーに対応するターゲット（プレイヤー）も表示したい。
3. 上記情報の追加に伴いレーン幅が不足する。

## 修正箇所
- `src/components/Timeline/SpawnMarker.tsx` — 方面名解決 + ターゲット表示
- `src/components/Timeline/GrillSlotLane.tsx` — SpawnMarker への props 追加
- `src/components/Timeline/index.tsx` — 湧き通し番号の計算、props の受け渡し
- `src/components/Timeline/coordinates.ts` — LANE_WIDTH 拡大
- `src/ScenarioView.tsx` — Timeline への props 追加

## 修正内容

### 1. 方面名の修正

SpawnMarker に `directionPresets` を渡し、`spawn.direction`（DirectionId）から表示名を解決する:

```tsx
interface SpawnMarkerProps {
  spawn: SpawnPoint;
  directionName: string;         // 解決済みの方面名
  targetLabel?: string | null;   // ターゲットのラベル（後述）
  targetIcon?: string | null;    // ターゲットのアイコンURL（後述）
  displayMode: DisplayMode;      // 表示モード
}
```

ラベル表示:
```tsx
<span ...>
  {seconds}s {directionName}
  {targetDisplay && <> {targetDisplay}</>}
</span>
```

### 2. ターゲットの決定ロジック

ターゲット順は **A枠B枠合わせた湧きの通し番号** で決定する。

#### 湧きの通し番号の計算

全湧き点（A枠+B枠）を frameTime の降順（時間的に早い順 = 大きい値が先）にソートし、1から連番を振る:

```ts
// Timeline index.tsx で計算
const spawnOrder: Map<string, number> = useMemo(() => {
  const allSpawns = [...spawns]
    .filter((s) => s.frameTime > 0)
    .sort((a, b) => b.frameTime - a.frameTime);
  const order = new Map<string, number>();
  allSpawns.forEach((spawn, i) => {
    order.set(spawn.id, i); // 0-indexed
  });
  return order;
}, [spawns]);
```

#### ターゲットの解決

```ts
const targetOrder = state.memo.targetOrder.order; // string[]（25要素、'1P'|'2P'|'3P'|'4P'|'-'）

// 各湧き点に対して:
const orderIndex = spawnOrder.get(spawn.id); // 0-indexed
const targetEntry = orderIndex != null ? targetOrder[orderIndex] : undefined;
// targetEntry が '1P'~'4P' ならターゲット確定、'-' or undefined なら表示しない
```

#### ターゲットの表示内容（DisplayMode に従う）

表示モード `state.displayMode` に応じて:

- `'icon'`: ブキアイコンのみ（16x16）
- `'text'`: 「1P」等のテキストのみ
- `'both'`: テキスト + ブキアイコン

ブキアイコンは `state.memo.weapons` からプレイヤーインデックスで引く:

```ts
const playerIndex = parseInt(targetEntry[0]!) - 1; // '1P' → 0
const weaponRowId = state.memo.weapons[playerIndex];
const weapon = weapons.find((w) => w.rowId === weaponRowId);
const iconUrl = weapon ? getWeaponIconPath(weapon.id) : null;
```

### 3. SpawnMarker の表示

```tsx
<span className="..." style={{ ... }}>
  {seconds}s {directionName}
  {/* ターゲット表示 */}
  {targetLabel && displayMode !== 'icon' && (
    <> {targetLabel}</>
  )}
  {targetIcon && displayMode !== 'text' && (
    <img src={targetIcon} alt="" className="inline-block" style={{ width: 14, height: 14, marginLeft: 2, verticalAlign: 'middle' }} />
  )}
</span>
```

### 4. レーン幅の拡大

現在 `LANE_WIDTH = 100` だが、ラベルに「95.8s 正面 1P🔫」のような情報が入るようになるため、ラベルがレーン外にはみ出す量が増える。

ラベルは既に `overflow: visible` で表示されており、レーン幅を超えてはみ出せるが、操作説明カードとの干渉を避けるため LANE_WIDTH を `120` 程度に拡大する。

また操作説明カードの `right` 位置もレーン幅拡大に合わせて調整すること。

### 5. props の受け渡しチェーン

```
ScenarioView
  → Timeline（spawns, defeats, directions, hazardConfig, directionPresets, targetOrder, weapons, weaponMaster, displayMode）
    → GrillSlotLane（+ spawnTargetMap）
      → SpawnMarker（directionName, targetLabel, targetIcon, displayMode）
```

Timeline に渡す情報が増えるので、`spawnTargetMap` のような事前計算済みの Map を Timeline 内で作り、GrillSlotLane → SpawnMarker に渡す設計が望ましい。

```ts
// Timeline 内で計算
interface SpawnDisplayInfo {
  directionName: string;
  targetLabel: string | null;    // '1P' 等、または null
  targetIcon: string | null;     // アイコンURL、または null
}

const spawnDisplayMap: Map<string, SpawnDisplayInfo> = useMemo(() => { ... }, [...]);
```

## 完了条件
- 湧きマーカー右のラベルで方面が「左」「正面」「右」等のプリセット名で表示されること
- ターゲット順が設定されている湧きには、対応するターゲット（プレイヤー名/ブキアイコン）が表示されること
- 表示モード（アイコン/テキスト/両方）に従ってターゲットの表示形式が切り替わること
- ターゲット順で '-'（未選択）の場合はターゲットが表示されないこと
- レーン幅が適切に拡大され、ラベルが他のUI要素と干渉しないこと
