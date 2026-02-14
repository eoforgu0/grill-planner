# 088: マーカー枠線を細い黒線に統一 + 抑制マーカーを斜線パターンに変更

## 種別
UI改善

## 概要

湧きマーカー・撃破マーカーの枠線を、フキダシ等と同じ細く薄い黒線（`1px solid var(--color-border)`）に統一する。
A枠/B枠のスロット色による枠線区別は廃止（レーン分離で区別可能なため）。

湧き抑制マーカーは斜線パターン（ストライプ）のデザインに変更する。

## 修正箇所

### SpawnMarker.tsx — 湧きマーカー

枠線の変更:

現在:
```tsx
border: `2px ${isSuppressed ? "dashed" : "solid"} ${borderColor}`,
```

変更後（通常）:
```tsx
border: "1px solid var(--color-border)",
```

変更後（抑制）: 枠線は同じ `1px solid var(--color-border)` だが、背景に斜線パターンを適用:
```tsx
{isSuppressed ? (
  <div
    className="shrink-0 rounded-full"
    style={{
      width: markerSize,
      height: markerSize,
      border: "1px solid var(--color-border)",
      background: `repeating-linear-gradient(
        -45deg,
        var(--color-spawn),
        var(--color-spawn) 2px,
        rgba(255,255,255,0.6) 2px,
        rgba(255,255,255,0.6) 4px
      )`,
    }}
  />
) : (
  <div
    className="shrink-0 rounded-full"
    style={{
      width: markerSize,
      height: markerSize,
      backgroundColor: "var(--color-spawn)",
      border: "1px solid var(--color-border)",
    }}
  />
)}
```

斜線パターン: 緑と白の45度ストライプ（2px幅交互）で、通常マーカーと明確に区別しつつ「抑制されている」ことを視覚的に伝える。

`borderColor` 変数（スロット色の参照）は不要になるため削除。

### DefeatMarker.tsx — 撃破マーカー

枠線の変更:

現在:
```tsx
border: `2px ${borderStyle} ${borderColor}`,
```

変更後:
```tsx
border: `1px solid var(--color-border)`,
```

ドラッグ中の無効状態（`isValidPosition === false`）:
```tsx
border: "1px dashed var(--color-border)",
```

`borderColor` 変数（スロット色の参照）は不要になるため削除。

### 凡例の更新（Timeline/index.tsx）

湧きマーカーの凡例:
```tsx
{/* 湧き */}
<div style={{
  width: 10, height: 10, borderRadius: "50%",
  backgroundColor: "var(--color-spawn)",
  border: "1px solid var(--color-border)",  // 変更
}} />

{/* 湧き（抑制） */}
<div style={{
  width: 10, height: 10, borderRadius: "50%",
  border: "1px solid var(--color-border)",  // 変更
  background: `repeating-linear-gradient(
    -45deg,
    var(--color-spawn), var(--color-spawn) 2px,
    rgba(255,255,255,0.6) 2px, rgba(255,255,255,0.6) 4px
  )`,
}} />
```

## 不要になるもの

- SpawnMarker / DefeatMarker 内の `borderColor` 変数（`slot === "A" ? "var(--color-slot-a)" : "var(--color-slot-b)"` の判定）
- 撃破マーカーの `borderStyle` 変数のうち、スロット色に関する分岐

## 完了条件
- 湧きマーカーの枠線が `1px solid var(--color-border)` になること
- 撃破マーカーの枠線が `1px solid var(--color-border)` になること
- 湧き抑制マーカーが斜線パターン（緑白ストライプ）で表示されること
- 凡例の表示がマーカーの実際の見た目と一致すること
- スロット色（赤/青）による枠線区別が廃止されていること
