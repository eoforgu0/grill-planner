# 086: 撃破マーカーの連動移動モード（Shift+ドラッグ / Shift+Enter確定）

## 種別
機能追加

## 概要

Shift キーを押しながら撃破マーカーをドラッグすると、同一枠の後続撃破マーカーも同じフレーム数だけ連動して移動する（連動モード）。Shift なしは従来通り個別移動（個別モード）。

数値直接入力（撃破時刻 / 湧き〜撃破間の経過時間）でも、Shift+Enter で確定すると連動移動する。

## 操作仕様

### 1. ドラッグ操作

- **Shift なしドラッグ**: 個別モード（従来動作）
- **Shift ありドラッグ**: 連動モード
- モード判定はドラッグ開始時点（DRAG_THRESHOLD 超過時点）の `shiftKey` で確定
- ドラッグ中の Shift 押下/解除では切り替わらない

### 2. 数値直接入力

#### DefeatMarker（撃破時刻入力）
- **Enter**: 個別移動（従来動作）
- **Shift+Enter**: 連動移動（delta = newFrameTime - oldFrameTime を後続撃破にも適用）

#### ElapsedTimeLabel（湧き〜撃破間の経過時間入力）
- **Enter**: 個別移動（従来動作）
- **Shift+Enter**: 連動移動（delta = newDefeatFrame - oldDefeatFrame を後続撃破にも適用）

### 3. 連動の移動対象

ドラッグ/編集対象の撃破より**時間的に未来**（frameTime が小さい）の**同一枠**の撃破すべて。

例: A枠で撃破が frameTime 降順で [D1(5700), D2(5200), D3(4700)] のとき:
- D1 を連動 → D2, D3 も連動
- D2 を連動 → D3 のみ連動
- D3 を連動 → 連動対象なし（個別と同じ）

### 4. 連動の計算

```
delta = newFrameTime - originalFrameTime  // ドラッグ対象の移動量（フレーム単位）
各連動撃破の新frameTime = 元のframeTime + delta
```

### 5. Shift ホバー時の視覚ヒント

撃破マーカーにホバーした状態で Shift を押す（または Shift を押しながらホバーする）と、
連動対象の撃破マーカーがハイライト表示される（例: マーカーの輪郭を太く or 色を変える）。

これにより「Shift+ドラッグしたらどの撃破が動くか」をドラッグ前に確認できる。

### 6. ドラッグ中の連動プレビュー

連動モードのドラッグ中、連動対象の撃破マーカーは移動先の位置にプレビュー表示される。
プレビューは半透明（opacity: 0.5）で表示し、ドラッグ中の撃破マーカーと区別する。

### 7. バリデーション

連動移動は全体を一括でバリデーションする。

ドラッグ中プレビュー: 簡易チェック（各連動撃破の newFrameTime が 0 < f < 6000 の範囲内）で十分。

確定時: `MOVE_DEFEATS_BATCH` で一括更新後、`calculateSpawns` が再実行されて全体整合性が保たれる。
確定前の最終バリデーションとして、仮の defeats 配列を作り `calculateSpawns` を再実行して
各撃破が対応する湧きより未来であることを検証する。

いずれか1つでもバリデーション失敗なら、全体を無効（`isValidPosition = false`、赤表示）とする。

## 修正箇所

### `src/hooks/useTimelineDrag.ts`

#### DragState 拡張

```ts
export interface LinkedDefeatPreview {
  defeatId: string;
  originalFrameTime: FrameTime;
  newFrameTime: FrameTime;
}

export interface DragState {
  isDragging: boolean;
  dragDefeatId: string | null;
  dragFrameTime: FrameTime | null;
  isValidPosition: boolean;
  isLinkedMode: boolean;
  linkedDefeats: readonly LinkedDefeatPreview[];
}
```

INITIAL_STATE に `isLinkedMode: false`, `linkedDefeats: []` を追加。

#### 引数追加

```ts
export function useTimelineDrag(
  pixelYToFrame: (pixelY: number) => FrameTime,
  validatePosition: (defeatId: string, frameTime: FrameTime) => boolean,
  validateLinkedMove: (moves: ReadonlyArray<{ defeatId: string; frameTime: FrameTime }>) => boolean,
  onDragEnd: (defeatId: string, frameTime: FrameTime) => void,
  onLinkedDragEnd: (moves: ReadonlyArray<{ defeatId: string; frameTime: FrameTime }>) => void,
  getLinkedDefeats: (defeatId: string) => DefeatPoint[],
  laneRef: React.RefObject<HTMLDivElement | null>,
)
```

#### candidateRef にshiftKeyとoriginalFrameTimeを追加

```ts
const candidateRef = useRef<{
  defeatId: string;
  startY: number;
  shiftKey: boolean;
  originalFrameTime: FrameTime;
} | null>(null);
```

`startDragCandidate` のシグネチャ:
```ts
startDragCandidate: (defeatId: string, startY: number, shiftKey: boolean, originalFrameTime: FrameTime) => void
```

#### handleMouseMove 内の連動計算

```ts
if (candidate.shiftKey) {
  const linked = getLinkedDefeatsRef.current(candidate.defeatId);
  const delta = frameTime - candidate.originalFrameTime;
  const linkedPreviews: LinkedDefeatPreview[] = linked.map(d => ({
    defeatId: d.id,
    originalFrameTime: d.frameTime,
    newFrameTime: d.frameTime + delta,
  }));

  const allMoves = [
    { defeatId: candidate.defeatId, frameTime },
    ...linkedPreviews.map(lp => ({ defeatId: lp.defeatId, frameTime: lp.newFrameTime })),
  ];
  const isValid = validateLinkedMoveRef.current(allMoves);

  setDragState({
    isDragging: true,
    dragDefeatId: candidate.defeatId,
    dragFrameTime: frameTime,
    isValidPosition: isValid,
    isLinkedMode: true,
    linkedDefeats: linkedPreviews,
  });
} else {
  // 従来の個別モード
  setDragState({
    isDragging: true,
    dragDefeatId: candidate.defeatId,
    dragFrameTime: frameTime,
    isValidPosition: isValid,
    isLinkedMode: false,
    linkedDefeats: [],
  });
}
```

#### handleMouseUp での連動確定

```ts
if (state.isLinkedMode && state.linkedDefeats.length > 0 && state.isValidPosition) {
  const allMoves = [
    { defeatId: candidate.defeatId, frameTime: state.dragFrameTime! },
    ...state.linkedDefeats.map(lp => ({ defeatId: lp.defeatId, frameTime: lp.newFrameTime })),
  ];
  onLinkedDragEndRef.current(allMoves);
} else if (!state.isLinkedMode && state.isValidPosition && state.dragFrameTime !== null) {
  onDragEndRef.current(candidate.defeatId, state.dragFrameTime);
}
```

### `src/components/Timeline/GrillSlotLane.tsx`

#### getLinkedDefeats

```tsx
const getLinkedDefeats = useCallback(
  (defeatId: string): DefeatPoint[] => {
    const target = slotDefeats.find(d => d.id === defeatId);
    if (!target) return [];
    return slotDefeats.filter(d => d.id !== defeatId && d.frameTime < target.frameTime);
  },
  [slotDefeats],
);
```

#### validateLinkedMove

```tsx
const validateLinkedMove = useCallback(
  (moves: ReadonlyArray<{ defeatId: string; frameTime: FrameTime }>): boolean => {
    return moves.every(m => {
      if (m.frameTime <= 0 || m.frameTime >= 6000) return false;
      return validateMoveDefeat?.(m.defeatId, m.frameTime) ?? true;
    });
  },
  [validateMoveDefeat],
);
```

#### handleLinkedDragEnd / handleLinkedTimeEdit

```tsx
const handleLinkedDragEnd = useCallback(
  (moves: ReadonlyArray<{ defeatId: string; frameTime: FrameTime }>) => {
    onLinkedMoveDefeats?.(moves);
  },
  [onLinkedMoveDefeats],
);
```

#### handleDefeatTimeEdit の拡張（Shift+Enter対応）

既存の `handleDefeatTimeEdit` を拡張して `isLinked` パラメータを受け取る:

```tsx
const handleDefeatTimeEdit = useCallback(
  (defeatId: string, newSeconds: number, isLinked: boolean): boolean => {
    const frameTime = secondsToFrames(newSeconds);
    const valid = validateMoveDefeat?.(defeatId, frameTime) ?? true;
    if (!valid) return false;

    if (isLinked) {
      const target = slotDefeats.find(d => d.id === defeatId);
      if (!target) return false;
      const delta = frameTime - target.frameTime;
      const linked = slotDefeats.filter(d => d.id !== defeatId && d.frameTime < target.frameTime);
      const allMoves = [
        { defeatId, frameTime },
        ...linked.map(d => ({ defeatId: d.id, frameTime: d.frameTime + delta })),
      ];
      // 全体バリデーション
      const allValid = allMoves.every(m => {
        if (m.frameTime <= 0 || m.frameTime >= 6000) return false;
        return validateMoveDefeat?.(m.defeatId, m.frameTime) ?? true;
      });
      if (!allValid) return false;
      onLinkedMoveDefeats?.(allMoves);
    } else {
      onMoveDefeat?.(defeatId, frameTime);
    }
    return true;
  },
  [validateMoveDefeat, onMoveDefeat, onLinkedMoveDefeats, slotDefeats],
);
```

#### Shift ホバーハイライト

```tsx
const [shiftHeld, setShiftHeld] = useState(false);
const [hoveredDefeatId, setHoveredDefeatId] = useState<string | null>(null);

useEffect(() => {
  const onKeyDown = (e: KeyboardEvent) => { if (e.key === "Shift") setShiftHeld(true); };
  const onKeyUp = (e: KeyboardEvent) => { if (e.key === "Shift") setShiftHeld(false); };
  window.addEventListener("keydown", onKeyDown);
  window.addEventListener("keyup", onKeyUp);
  return () => {
    window.removeEventListener("keydown", onKeyDown);
    window.removeEventListener("keyup", onKeyUp);
  };
}, []);

const linkedHighlightIds = useMemo(() => {
  if (!shiftHeld || !hoveredDefeatId) return new Set<string>();
  const target = slotDefeats.find(d => d.id === hoveredDefeatId);
  if (!target) return new Set<string>();
  return new Set(
    slotDefeats
      .filter(d => d.id !== hoveredDefeatId && d.frameTime < target.frameTime)
      .map(d => d.id)
  );
}, [shiftHeld, hoveredDefeatId, slotDefeats]);
```

#### useTimelineDrag の呼び出し更新

```tsx
const { dragState, startDragCandidate, justFinishedDragRef } = useTimelineDrag(
  scaledPixelYToFrameFn,
  validatePosition,
  validateLinkedMove,
  handleDragEnd,
  handleLinkedDragEnd,
  getLinkedDefeats,
  laneRef,
);
```

#### handleDefeatMouseDown の更新

```tsx
const handleDefeatMouseDown = useCallback(
  (defeatId: string, startY: number, shiftKey: boolean) => {
    const defeat = slotDefeats.find(d => d.id === defeatId);
    startDragCandidate(defeatId, startY, shiftKey, defeat?.frameTime ?? 0);
  },
  [startDragCandidate, slotDefeats],
);
```

#### DefeatMarker のレンダリング

```tsx
{slotDefeats.map((defeat) => {
  const isLinkedPreview = dragState.isLinkedMode
    && dragState.linkedDefeats.some(lp => lp.defeatId === defeat.id);
  const linkedPreviewFrame = isLinkedPreview
    ? dragState.linkedDefeats.find(lp => lp.defeatId === defeat.id)?.newFrameTime ?? null
    : null;

  return (
    <DefeatMarker
      key={defeat.id}
      defeat={defeat}
      isDragging={dragState.isDragging && dragState.dragDefeatId === defeat.id}
      dragFrameTime={dragState.dragDefeatId === defeat.id ? dragState.dragFrameTime : null}
      isValidPosition={dragState.dragDefeatId === defeat.id ? dragState.isValidPosition : true}
      isLinkedPreview={isLinkedPreview}
      linkedPreviewFrameTime={linkedPreviewFrame}
      isLinkedHighlight={linkedHighlightIds.has(defeat.id)}
      scaleX={scaleX}
      scaleY={scaleY}
      onMouseDown={handleDefeatMouseDown}
      onContextMenu={handleDefeatContextMenu}
      onTimeEdit={handleDefeatTimeEdit}
      onHoverChange={(hovered) => setHoveredDefeatId(hovered ? defeat.id : null)}
    />
  );
})}
```

### `src/components/Timeline/DefeatMarker.tsx`

#### props 追加

```ts
interface DefeatMarkerProps {
  // ...既存
  isLinkedPreview?: boolean;
  linkedPreviewFrameTime?: FrameTime | null;
  isLinkedHighlight?: boolean;
  onMouseDown?: (defeatId: string, startY: number, shiftKey: boolean) => void;
  onTimeEdit?: (defeatId: string, newSeconds: number, isLinked: boolean) => boolean;
  onHoverChange?: (hovered: boolean) => void;
}
```

#### handleMouseDown — shiftKey を渡す

```tsx
onMouseDown?.(defeat.id, e.clientY, e.shiftKey);
```

#### ホバーコールバック

```tsx
onMouseEnter={() => { setIsHovered(true); onHoverChange?.(true); }}
onMouseLeave={() => { setIsHovered(false); onHoverChange?.(false); }}
```

#### 連動ハイライト表示

`isLinkedHighlight` が true の場合、マーカーの枠線を太くする（例: `borderWidth: 3`）または背景色を薄く変化させる:

```tsx
const highlightStyle = isLinkedHighlight ? {
  boxShadow: "0 0 0 2px rgba(59, 130, 246, 0.4)",
} : {};
```

#### 連動プレビュー表示

`isLinkedPreview && linkedPreviewFrameTime != null` の場合、
マーカーを `linkedPreviewFrameTime` の位置に半透明（opacity: 0.5）で表示:

```tsx
const displayFrame = isLinkedPreview && linkedPreviewFrameTime != null
  ? linkedPreviewFrameTime
  : (isDragging && dragFrameTime != null ? dragFrameTime : defeat.frameTime);

const opacity = isLinkedPreview ? 0.5 : 1;
```

#### 時間確定で Shift+Enter 判定

```tsx
const handleKeyDown = useCallback(
  (e: KeyboardEvent<HTMLInputElement>) => {
    e.stopPropagation();
    if (e.key === "Enter") {
      confirmEdit(e.shiftKey);
    } else if (e.key === "Escape") {
      setEditing(false);
    }
  },
  [confirmEdit],
);

const confirmEdit = useCallback(
  (isLinked = false) => {
    setEditing(false);
    const newSeconds = parseFloat(editValue);
    if (Number.isNaN(newSeconds) || newSeconds < 0 || newSeconds > 100) return;
    if (newSeconds === seconds) return;
    onTimeEdit?.(defeat.id, newSeconds, isLinked);
  },
  [editValue, seconds, defeat.id, onTimeEdit],
);
```

onBlur では個別移動（`confirmEdit(false)`）。

### `src/components/Timeline/ElapsedTimeLabel.tsx`

#### onTimeEdit のシグネチャ変更

```ts
onTimeEdit?: (defeatId: string, newSeconds: number, isLinked: boolean) => boolean;
```

#### Shift+Enter 判定

```tsx
const handleKeyDown = useCallback(
  (e: KeyboardEvent<HTMLInputElement>) => {
    e.stopPropagation();
    if (e.key === "Enter") {
      confirmEdit(e.shiftKey);
    } else if (e.key === "Escape") {
      setEditing(false);
    }
  },
  [confirmEdit],
);

const confirmEdit = useCallback(
  (isLinked = false) => {
    setEditing(false);
    const inputSeconds = Number.parseFloat(editValue);
    if (Number.isNaN(inputSeconds) || inputSeconds < 0) return;
    const newDefeatSeconds = spawnSeconds - inputSeconds;
    if (newDefeatSeconds < 0 || newDefeatSeconds > 100) return;
    onTimeEdit?.(defeatId, newDefeatSeconds, isLinked);
  },
  [editValue, spawnSeconds, defeatId, onTimeEdit],
);
```

onBlur では個別移動（`confirmEdit(false)`）。

### `src/components/Timeline/index.tsx`

#### handleLinkedMoveDefeats

```tsx
const handleLinkedMoveDefeats = useCallback(
  (moves: ReadonlyArray<{ defeatId: string; frameTime: FrameTime }>) => {
    dispatch({ type: "MOVE_DEFEATS_BATCH", payload: moves.map(m => ({ id: m.defeatId, frameTime: m.frameTime })) });
  },
  [dispatch],
);
```

注: `MOVE_DEFEATS_BATCH` は既に scenarioReducer に実装済み。

GrillSlotLane に `onLinkedMoveDefeats={handleLinkedMoveDefeats}` を渡す。

#### 操作説明カードに追加

```
Shift+ドラッグ: 後続撃破も連動移動
Shift+Enter: 時間入力でも連動移動
```

### `src/components/Timeline/GrillSlotLane.tsx` — props 追加

```ts
interface GrillSlotLaneProps {
  // ...既存
  onLinkedMoveDefeats?: (moves: ReadonlyArray<{ defeatId: string; frameTime: FrameTime }>) => void;
}
```

## 完了条件

### ドラッグ操作
- Shift なしドラッグで従来通り個別移動すること
- Shift ありドラッグで同枠の後続撃破が連動して移動すること
- 連動ドラッグ中、連動対象の撃破マーカーが移動先に半透明でプレビュー表示されること
- 連動移動でバリデーション不正になる場合、ドラッグ中に赤表示されること
- 連動移動確定後、湧きが正しく再計算されること
- ESCキーで連動ドラッグをキャンセルできること

### 数値入力
- DefeatMarker の時刻入力で Enter → 個別移動、Shift+Enter → 連動移動すること
- ElapsedTimeLabel の経過時間入力で Enter → 個別移動、Shift+Enter → 連動移動すること
- onBlur（フォーカス外れ）では個別移動すること
- 連動移動でバリデーション不正の場合、移動が拒否されること

### 視覚ヒント
- 撃破マーカーにホバーした状態で Shift を押すと、連動対象の撃破マーカーがハイライトされること
- Shift を離すとハイライトが消えること
- ドラッグしていない撃破マーカーのハイライトが通常の操作に影響しないこと

### 操作説明
- 操作説明カードに「Shift+ドラッグ: 後続撃破も連動移動」が表示されること
- 操作説明カードに「Shift+Enter: 時間入力でも連動移動」が表示されること
