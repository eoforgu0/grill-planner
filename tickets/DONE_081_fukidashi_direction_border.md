# 081: フキダシの枠線を方面色にする

## 種別
UI改善

## 概要

湧きマーカー・撃破マーカーのフキダシ（時刻表示やテキスト表示の背景付きラベル）に、そのグリルが湧いた方面の色で細い枠線を付ける。

これにより、どの方面から湧いたグリルかがフキダシ部分からも視覚的にわかるようになる。

## 対象コンポーネント

### SpawnMarker.tsx — 湧きフキダシ

現在:
```tsx
<span style={{
  ...
  backgroundColor: "rgba(255,255,255,0.85)",
  padding: "1px 4px",
  borderRadius: 2,
}}>
```

変更後:
```tsx
<span style={{
  ...
  backgroundColor: "rgba(255,255,255,0.85)",
  padding: "1px 4px",
  borderRadius: 2,
  border: `1px solid ${directionColor}`,  // 追加
}}>
```

`directionColor` は `getDirectionColor(spawn.direction)` で取得。
SpawnMarker は既に `spawn.direction` を持っているので追加のpropsは不要。

ブキアイコンの独立背景にも同様に方面色枠線を適用:

現在:
```tsx
<div style={{
  ...
  border: "1px solid var(--color-border)",
}}>
```

変更後:
```tsx
<div style={{
  ...
  border: `1px solid ${directionColor}`,
}}>
```

### DefeatMarker.tsx — 撃破フキダシ

撃破マーカーは自身の `direction` を持っていないため、対応する湧きの方面を知る必要がある。

**方法**: GrillSlotLane から DefeatMarker に `directionColor` を props で渡す。

GrillSlotLane の `spawnDefeatPairs` を活用して、各撃破に対応する湧きの方面を特定:

```tsx
// GrillSlotLane.tsx
const defeatDirectionMap = useMemo(() => {
  const map = new Map<string, DirectionId>();
  for (const { spawn, defeat } of spawnDefeatPairs) {
    if (defeat) {
      map.set(defeat.id, spawn.direction);
    }
  }
  return map;
}, [spawnDefeatPairs]);
```

DefeatMarker に props 追加:
```tsx
interface DefeatMarkerProps {
  ...
  directionColor?: string;  // 追加
}
```

撃破フキダシ（表示モード）:

現在:
```tsx
<span style={{
  ...
  backgroundColor: "rgba(255,255,255,0.85)",
  padding: "1px 4px",
  borderRadius: 2,
}}>
```

変更後:
```tsx
<span style={{
  ...
  backgroundColor: "rgba(255,255,255,0.85)",
  padding: "1px 4px",
  borderRadius: 2,
  border: directionColor ? `1px solid ${directionColor}` : undefined,  // 追加
}}>
```

編集モードの input にも同様に枠線色を適用:
```tsx
<input
  style={{
    ...
    border: directionColor ? `1px solid ${directionColor}` : "1px solid var(--color-border)",
  }}
/>
```

## ElapsedTimeLabel.tsx（チケット080）

チケット080で新規作成する ElapsedTimeLabel は、設計時点で方面色枠線を含んでいるため追加変更不要。

## 修正箇所

- `src/components/Timeline/SpawnMarker.tsx` — フキダシとブキアイコンに方面色枠線追加
- `src/components/Timeline/DefeatMarker.tsx` — `directionColor` props 追加、フキダシに枠線追加
- `src/components/Timeline/GrillSlotLane.tsx` — `defeatDirectionMap` を作成し DefeatMarker に `directionColor` を渡す
- `src/components/Timeline/coordinates.ts` — `getDirectionColor` のインポートが GrillSlotLane で使えることを確認

## 注意: 方面色の見え方

カラーテーマ（花・パステル・霞）の色は淡い背景色として設計されているため、枠線として使うと見えにくい可能性がある。

対策:
- 枠線にはカラーテーマの色をそのまま使い、背景色の `rgba(255,255,255,0.85)` との組み合わせで十分なコントラストがあるかを確認
- 必要に応じてカラーテーマとは別に「方面枠線色」を定義する可能性あり（ただしまずはそのまま試す）

## 完了条件
- 湧きマーカーのフキダシに方面色の枠線（1px solid）が表示されること
- ブキアイコンの背景枠線が方面色になること
- 撃破マーカーのフキダシに、対応する湧きの方面色の枠線が表示されること
- 撃破マーカーの編集モードのinputにも方面色の枠線が適用されること
- カラーテーマ変更時に枠線色が連動すること
