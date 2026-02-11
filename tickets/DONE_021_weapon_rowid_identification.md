# 021: ブキ/スペシャルの識別に RowId を使用する（Id=0 のボールドマーカーが選択できない問題）

## 種別
バグ

## 問題
ブキを選択する際、ボールドマーカー（Id=0）を選択しても選択状態にならない。

原因: `MemoSection.tsx` のブキ/スペシャル選択で `select` の `value` と `option` の `value` に数値の `Id` を使用しており、未選択状態を `0` で表現している:

```tsx
const selectedId = memo.weapons[i] ?? 0;
// ...
<option value={0}>--</option>
{weapons.map((w) => (
  <option key={w.id} value={w.id}>{w.label}</option>
))}
```

ボールドマーカーの `Id = 0` が未選択の `0` と衝突するため、選択しても「未選択」と判定される。

## 修正内容
ブキ/スペシャルの識別に `RowId`（文字列）を使用する。`RowId` は各ブキ/スペシャルに固有の文字列であり、未選択状態との衝突が起きない。

### 1. ScenarioMemo の型変更

```ts
// src/types/scenario.ts
export interface ScenarioMemo {
  readonly scenarioCode: string;
  readonly weapons: readonly string[];    // number[] → string[] (RowId)
  readonly specials: readonly string[];   // number[] → string[] (RowId)
  readonly targetOrder: { ... };
  readonly snatchers: string;
}
```

### 2. scenarioReducer のアクション変更

```ts
| { type: 'SET_WEAPON'; payload: { index: number; rowId: string } }
| { type: 'SET_SPECIAL'; payload: { index: number; rowId: string } }
```

### 3. MemoSection の修正

```tsx
const selectedRowId = memo.weapons[i] ?? '';
// ...
<option value="">--</option>
{weapons.map((w) => (
  <option key={w.rowId} value={w.rowId}>{w.label}</option>
))}
```

未選択状態は空文字列 `''` で表現する。

### 4. アイコンパス生成の修正

アイコン画像ファイル名は数値 Id に基づいている（`0.png`, `10.png` 等）ため、RowId → Id の逆引きが必要。MemoSection 内で weapons/specials の master データから RowId で lookup して Id を取得し、アイコンパスを生成する:

```ts
const weapon = weapons.find((w) => w.rowId === selectedRowId);
if (weapon) {
  const iconPath = getWeaponIconPath(weapon.id);
}
```

### 5. fileIO.ts のインポート互換性

保存済みファイルとの互換性に注意。既存の保存ファイルには `weapons: number[]` が含まれている可能性があるため、インポート時に number → RowId への変換を行う:

```ts
// number の場合は weapons master から RowId に変換
if (typeof weapon === 'number') {
  const found = weaponsMaster.find((w) => w.id === weapon);
  return found?.rowId ?? '';
}
```

### 6. ScenarioView の props 変更

`handleSetWeapon` / `handleSetSpecial` のシグネチャを `(index: number, rowId: string)` に変更。

## 修正箇所
- `src/types/scenario.ts`
- `src/hooks/scenarioReducer.ts`
- `src/components/Settings/MemoSection.tsx`
- `src/ScenarioView.tsx`
- `src/utils/fileIO.ts`

## 完了条件
- ボールドマーカー（Id=0）が正常に選択・表示されること
- 全ブキ・全スペシャルが選択可能なこと
- 選択したブキ/スペシャルのアイコンが正しく表示されること
- 未選択状態が `--` で表示され、いずれのブキ/スペシャルとも衝突しないこと
- 既存の保存ファイル（number[] 形式）がインポート可能なこと
