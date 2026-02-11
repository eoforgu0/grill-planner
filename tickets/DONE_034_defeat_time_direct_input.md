# 025: 撃破マーカーの時間ラベルクリックで撃破時間を直接入力して移動できるようにする

## 種別
UI改善（機能追加）

## 問題
撃破マーカーの移動手段がドラッグのみ。細かい時間指定が難しい。時間ラベルをクリックして直接入力で移動したい。

## 修正箇所
- `src/components/Timeline/DefeatMarker.tsx` — 時間ラベルのクリックで編集モードに入る機能
- `src/components/Timeline/GrillSlotLane.tsx` — 時間入力確定時のバリデーション + 移動処理のコールバック追加

## 修正内容

### DefeatMarker に編集モードを追加

1. 時間ラベル（`95.8s` 等）をクリックすると、ラベルが input に変化する（インライン編集）
2. input は `type="number"` で、step=0.1、min=0、max=100
3. 表示は秒単位（小数1桁）。入力も秒単位。
4. Enter で確定 → 秒をフレームに変換 → バリデーション → 移動
5. Escape でキャンセル
6. blur で確定
7. バリデーション失敗時は元の位置に戻す（入力をリセット）

### イベント伝播の制御

- 時間ラベルの click で `e.stopPropagation()` して、レーンの onClick（撃破追加）やマーカーの onMouseDown（ドラッグ開始）と干渉しないようにする
- 編集モード中の mousedown/click も stopPropagation する

### バリデーション

既存の `validateMoveDefeat` を使用する。GrillSlotLane から DefeatMarker に `onTimeEdit` コールバックを渡す:

```tsx
// GrillSlotLane.tsx
const handleDefeatTimeEdit = useCallback(
  (defeatId: string, newSeconds: number) => {
    const frameTime = secondsToFrames(newSeconds);
    const result = canMoveDefeat(defeatId, frameTime);
    if (result.valid) {
      // getAffectedDefeats + REMOVE_DEFEATS + MOVE_DEFEAT（handleMoveDefeat と同等）
      onMoveDefeat?.(defeatId, frameTime);
      return true;
    }
    return false;
  },
  [canMoveDefeat, onMoveDefeat],
);
```

### DefeatMarker の props 追加

```tsx
interface DefeatMarkerProps {
  // ...既存props
  onTimeEdit?: (defeatId: string, newSeconds: number) => boolean;
}
```

### input のスタイル

```tsx
<input
  type="number"
  step={0.1}
  min={0}
  max={100}
  className="w-12 rounded border border-border bg-surface px-1 text-center"
  style={{ fontSize: 9 }}
/>
```

## 完了条件
- 撃破マーカーの時間ラベルをクリックすると入力欄に変わること
- 秒数を入力して Enter で確定すると、撃破が指定時間に移動すること
- バリデーションが通らない場合は移動されず、元の値に戻ること
- Escape でキャンセルできること
- 入力中に他の操作（ドラッグ等）と干渉しないこと
