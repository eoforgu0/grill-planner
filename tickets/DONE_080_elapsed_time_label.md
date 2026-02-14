# 080: 湧き→撃破間の経過時間フキダシ表示（左側、編集可能）

## 種別
機能追加

## 概要

湧きマーカーと撃破マーカーの中間位置に、「湧いてから撃破するまでの経過時間」をフキダシで表示する。
フキダシは**左側**に配置（右側は湧きマーカーのフキダシ等で混み合うため）。
フキダシのクリックで時間を直接入力でき、入力値に基づいて撃破マーカーを移動する。

## 表示仕様

### 位置
- Y座標: 湧きマーカーと撃破マーカーの中間（`(spawnPixelY + defeatPixelY) / 2`）
- X座標: マーカー中心線（`MARKER_CENTER_RATIO` の位置）の**左側**
- フキダシは右寄せ（マーカー中心線に接する形で左に展開）

### 表示内容
- `{経過秒数}s` 形式（例: `3.6s`）
- 経過秒数 = `framesToSeconds(spawnFrame) - framesToSeconds(defeatFrame)`
  （フレーム値が大きい方が過去 = 湧きが過去、撃破が未来なので湧き秒 - 撃破秒 = 正の値）

### 編集機能
- フキダシをクリックすると入力モードに切り替わる（DefeatMarker と同じ仕組み）
- 入力した秒数は「湧きからの経過時間」として解釈
- 確定時: `newDefeatFrame = spawnFrame - secondsToFrames(inputSeconds)`
  - spawnFrame は当該湧きのframeTime（抑制適用済みの実際の値）
- バリデーション: `validateMoveDefeat` を通す。不正なら変更を破棄

## 新規コンポーネント

`src/components/Timeline/ElapsedTimeLabel.tsx`

```tsx
interface ElapsedTimeLabelProps {
  spawnFrame: FrameTime;
  defeatFrame: FrameTime;
  defeatId: string;
  directionColor: string;  // 方面色（枠線色に使用）
  scaleX: number;
  scaleY: number;
  onTimeEdit?: (defeatId: string, newSeconds: number) => boolean;
}
```

### 表示モード
```tsx
<div
  className="absolute flex items-center justify-end"
  style={{
    top: midY,
    right: `${(1 - MARKER_CENTER_RATIO) * 100}%`,  // マーカー中心線の左
    transform: "translateY(-50%)",
    zIndex: 3,
  }}
>
  <span
    className="cursor-text select-none whitespace-nowrap"
    style={{
      marginRight: 4,
      fontSize,
      color: "var(--color-text-muted)",
      backgroundColor: "rgba(255,255,255,0.85)",
      padding: "1px 4px",
      borderRadius: 2,
      border: `1px solid ${directionColor}`,  // 方面色の枠線
    }}
    onClick={startEdit}
  >
    {elapsed}s
  </span>
</div>
```

### 編集モード
```tsx
<input
  type="number"
  step={0.1}
  min={0}
  value={editValue}
  onChange={...}
  onKeyDown={...}  // Enter で確定、Escape でキャンセル
  onBlur={confirmEdit}
  style={{
    width: 48,
    marginRight: 4,
    fontSize,
    textAlign: "center",
    border: `1px solid ${directionColor}`,
  }}
/>
```

### 確定処理
```ts
const confirmEdit = () => {
  const inputSeconds = parseFloat(editValue);
  if (Number.isNaN(inputSeconds) || inputSeconds < 0) return;
  // 湧きフレームから経過時間分だけ未来（フレーム値が小さい方向）に撃破
  const spawnSeconds = framesToSeconds(spawnFrame);
  const newDefeatSeconds = spawnSeconds - inputSeconds;
  if (newDefeatSeconds < 0 || newDefeatSeconds > 100) return;
  onTimeEdit?.(defeatId, newDefeatSeconds);
};
```

## GrillSlotLane での組み込み

`spawnDefeatPairs` を使って、湧き-撃破ペアに対して `ElapsedTimeLabel` をレンダリング:

```tsx
{/* 経過時間ラベル */}
{spawnDefeatPairs
  .filter(({ spawn, defeat }) => spawn.frameTime > 0 && defeat !== null)
  .map(({ spawn, defeat }) => {
    const displayInfo = spawnDisplayMap.get(spawn.id);
    const directionColor = getDirectionColor(spawn.direction);
    return (
      <ElapsedTimeLabel
        key={`elapsed-${spawn.id}`}
        spawnFrame={spawn.frameTime}
        defeatFrame={defeat.frameTime}
        defeatId={defeat.id}
        directionColor={directionColor}
        scaleX={scaleX}
        scaleY={scaleY}
        onTimeEdit={handleDefeatTimeEdit}
      />
    );
  })}
```

## 修正箇所

### 新規ファイル
- `src/components/Timeline/ElapsedTimeLabel.tsx`

### 修正ファイル
- `src/components/Timeline/GrillSlotLane.tsx` — `ElapsedTimeLabel` のレンダリング追加
- `src/components/Timeline/coordinates.ts` — `getDirectionColor` のインポート追加（GrillSlotLane から使えるよう）

## 完了条件
- 湧き-撃破ペアの中間位置に経過時間がフキダシで表示されること
- フキダシがマーカー中心線の**左側**に配置されること
- フキダシの枠線が方面色であること
- フキダシクリックで入力モードに切り替わること
- 経過時間の入力で撃破マーカーが移動すること（湧き基準）
- バリデーションが通らない入力は破棄されること
- Enter で確定、Escape でキャンセルできること
- ドラッグ中の撃破マーカーに対しては経過時間が動的に更新されること（ドラッグ中の表示は任意、非表示でも可）
