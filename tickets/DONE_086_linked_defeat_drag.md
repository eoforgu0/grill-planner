# 086: 撃破マーカーの連動移動モード（Shift+ドラッグ）

## 種別
機能追加

## 概要

Shift キーを押しながら撃破マーカーをドラッグすると、同一枠の後続撃破マーカーも同じフレーム数だけ連動して移動する。Shift なしのドラッグは従来通り個別移動（個別モード）。

## 操作仕様

### ドラッグ開始時
- Shift **なし**: 個別モード（従来動作、変更なし）
- Shift **あり**: 連動モード

モード判定はドラッグ開始時点（mousedown → DRAG_THRESHOLD 超過時点）の `e.shiftKey` で確定する。
ドラッグ中の Shift 押下/解除では切り替わらない（案C方式）。

### 連動モードの移動対象

ドラッグ対象の撃破より**時間的に未来**（frameTime が小さい）の同一枠の撃破すべて。

例: A枠で撃破が frameTime 降順で [D1(5700), D2(5200), D3(4700)] のとき:
- D1 を連動ドラッグ → D2, D3 も連動
- D2 を連動ドラッグ → D3 のみ連動
- D3 を連動ドラッグ → 連動対象なし（個別と同じ）

### 連動の計算

```
delta = newFrameTime - originalFrameTime  // ドラッグ対象の移動量
各連動撃破の新frameTime = 元のframeTime + delta
```

### Shiftキー押下時の視覚ヒント

Shift を押している間（ドラッグ中かどうかに関わらず）、ホバー中の撃破マーカーがあれば、
連動対象の撃破マーカーをハイライト表示する（枠線を太くする、または背景色を変える等）。

→ これはドラッグ開始前に「連動するとどれが動くか」を確認できるようにするための視覚フィードバック。

実装が複雑になりすぎるなら、この視覚ヒントは省略して操作説明カードの説明だけでもよい。

### バリデーション

連動移動時は**全ての連動撃破を含めて**バリデーションを行う必要がある:

1. ドラッグ対象の撃破が有効位置か
2. 各連動撃破が有効位置か（移動後の位置で、再計算された湧きより未来であること）

いずれか1つでもバリデーション失敗なら、全体を無効とする（`isValidPosition = false`）。

### ドラッグ中のプレビュー表示

連動モードのドラッグ中:
- ドラッグ対象の撃破: 従来通りドラッグプレビュー表示
- 連動対象の撃破: 移動後の位置にプレビュー表示（個別モードの撃破マーカーと区別するため、半透明や破線などの表示にする）

→ 実装が複雑になる場合は、ドラッグ中の連動プレビューは省略してもよい。
   その場合、ドロップ時に一括で位置が更新される。

## 修正箇所

### `src/hooks/useTimelineDrag.ts`

**DragState に連動情報を追加:**

```ts
export interface DragState {
  isDragging: boolean;
  dragDefeatId: string | null;
  dragFrameTime: FrameTime | null;
  isValidPosition: boolean;
  isLinkedMode: boolean;                          // 追加: 連動モードか
  linkedDefeats: readonly LinkedDefeatPreview[];   // 追加: 連動撃破のプレビュー情報
}

export interface LinkedDefeatPreview {
  defeatId: string;
  originalFrameTime: FrameTime;
  newFrameTime: FrameTime;
}
```

**startDragCandidate に shiftKey を渡す:**

```ts
const startDragCandidate = useCallback((defeatId: string, startY: number, shiftKey: boolean) => {
  candidateRef.current = { defeatId, startY, shiftKey };
}, []);
```

**連動撃破リストの取得:**

useTimelineDrag に同一枠の撃破リスト（frameTime降順ソート済み）を渡す必要がある。
新しい引数として `getLinkedDefeats: (defeatId: string) => DefeatPoint[]` コールバックを追加:

```ts
export function useTimelineDrag(
  pixelYToFrame: (pixelY: number) => FrameTime,
  validatePosition: (defeatId: string, frameTime: FrameTime) => boolean,
  validateLinkedMove: (moves: ReadonlyArray<{ defeatId: string; frameTime: FrameTime }>) => boolean,  // 追加
  onDragEnd: (defeatId: string, frameTime: FrameTime) => void,
  onLinkedDragEnd: (moves: ReadonlyArray<{ defeatId: string; frameTime: FrameTime }>) => void,  // 追加
  getLinkedDefeats: (defeatId: string) => DefeatPoint[],  // 追加
  laneRef: React.RefObject<HTMLDivElement | null>,
)
```

**handleMouseMove 内の連動計算:**

```ts
if (candidate.shiftKey) {
  // 連動モード
  const linked = getLinkedDefeatsRef.current(candidate.defeatId);
  const delta = frameTime - originalFrameTimeRef.current;  // originalFrameTimeも保持が必要
  const linkedPreviews = linked.map(d => ({
    defeatId: d.id,
    originalFrameTime: d.frameTime,
    newFrameTime: d.frameTime + delta,
  }));

  // 全体バリデーション
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
  // 個別モード（従来）
  // ...
}
```

**handleMouseUp での連動確定:**

```ts
if (state.isLinkedMode && state.linkedDefeats.length > 0) {
  const allMoves = [
    { defeatId: candidate.defeatId, frameTime: state.dragFrameTime },
    ...state.linkedDefeats.map(lp => ({ defeatId: lp.defeatId, frameTime: lp.newFrameTime })),
  ];
  onLinkedDragEndRef.current(allMoves);
} else {
  onDragEndRef.current(candidate.defeatId, state.dragFrameTime);
}
```

### `src/components/Timeline/GrillSlotLane.tsx`

**getLinkedDefeats コールバック:**

```tsx
const getLinkedDefeats = useCallback(
  (defeatId: string): DefeatPoint[] => {
    const target = slotDefeats.find(d => d.id === defeatId);
    if (!target) return [];
    // frameTimeが小さい（時間的に未来の）同枠撃破を返す
    return slotDefeats.filter(d => d.id !== defeatId && d.frameTime < target.frameTime);
  },
  [slotDefeats],
);
```

**onLinkedDragEnd ハンドラ:**

連動移動の確定では複数の撃破を一括で移動する。

```tsx
const handleLinkedDragEnd = useCallback(
  (moves: ReadonlyArray<{ defeatId: string; frameTime: FrameTime }>) => {
    onLinkedMoveDefeats?.(moves);
  },
  [onLinkedMoveDefeats],
);
```

**DefeatMarker への Shift ホバーヒント:**

Shift 押下中にホバーしている撃破マーカーの連動対象をハイライトするには、
GrillSlotLane レベルで Shift キー状態を追跡し、各 DefeatMarker に「連動対象か」を伝える必要がある。

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

// 各 DefeatMarker に isLinkedHighlight を渡す
const linkedTargetIds = useMemo(() => {
  if (!shiftHeld || !hoveredDefeatId) return new Set<string>();
  const target = slotDefeats.find(d => d.id === hoveredDefeatId);
  if (!target) return new Set<string>();
  return new Set(slotDefeats.filter(d => d.id !== hoveredDefeatId && d.frameTime < target.frameTime).map(d => d.id));
}, [shiftHeld, hoveredDefeatId, slotDefeats]);
```

→ 実装が複雑になるなら省略してよい。

**handleDefeatMouseDown の修正:**

```tsx
const handleDefeatMouseDown = useCallback(
  (defeatId: string, startY: number, shiftKey: boolean) => {
    startDragCandidate(defeatId, startY, shiftKey);
  },
  [startDragCandidate],
);
```

### `src/components/Timeline/DefeatMarker.tsx`

**onMouseDown で shiftKey を渡す:**

```tsx
const handleMouseDown = useCallback(
  (e: MouseEvent) => {
    if (editing) return;
    if (e.button !== 0) return;
    e.stopPropagation();
    onMouseDown?.(defeat.id, e.clientY, e.shiftKey);  // shiftKey 追加
  },
  [defeat.id, onMouseDown, editing],
);
```

**連動ドラッグ中のプレビュー表示（オプション）:**

`isLinkedDragPreview` props を追加し、連動対象の撃破を移動先位置に半透明で表示:

```tsx
interface DefeatMarkerProps {
  ...
  isLinkedDragPreview?: boolean;
  linkedDragFrameTime?: FrameTime | null;
}
```

### `src/components/Timeline/index.tsx`

**onLinkedMoveDefeats ハンドラ:**

```tsx
const handleLinkedMoveDefeats = useCallback(
  (moves: ReadonlyArray<{ defeatId: string; frameTime: FrameTime }>) => {
    // 各撃破を順に移動（後続への影響は calculateSpawns で自動処理）
    for (const move of moves) {
      // 移動先が不正な撃破を事前除去
      const affected = getAffectedDefeats(move.defeatId, move.frameTime, defeats);
      if (affected.length > 0) {
        dispatch({ type: "REMOVE_DEFEATS", payload: affected.map(d => d.id) });
      }
      dispatch({ type: "MOVE_DEFEAT", payload: { id: move.defeatId, frameTime: move.frameTime } });
    }
  },
  [defeats, dispatch],
);
```

注意: 複数の dispatch を連続で呼ぶと中間状態が発生する。
これを避けるには reducer に `MOVE_DEFEATS_BATCH` アクションを追加して一括更新するのが望ましい。

```ts
// scenarioReducer に追加
case "MOVE_DEFEATS_BATCH": {
  const moves = new Map(action.payload.map(m => [m.id, m.frameTime]));
  return {
    ...state,
    defeats: state.defeats.map(d => {
      const newFrame = moves.get(d.id);
      return newFrame !== undefined ? { ...d, frameTime: newFrame } : d;
    }),
  };
}
```

**validateLinkedMove:**

```tsx
const validateLinkedMove = useCallback(
  (moves: ReadonlyArray<{ defeatId: string; frameTime: FrameTime }>): boolean => {
    // 全ての移動先が個別にバリデーションを通るか
    // ※ 厳密には相互依存があるが、同枠の撃破チェーンは先頭から順に依存するため、
    //    frameTime降順（過去→未来）で検証すれば正しい
    return moves.every(m => canMoveDefeat(m.defeatId, m.frameTime).valid);
  },
  [canMoveDefeat],
);
```

→ ただし canMoveDefeat は現在の spawns を基準にバリデーションするため、
  連動移動で先頭の撃破が動いた後の湧きを考慮できない。
  
  簡易的には「ドラッグ対象の撃破のバリデーションのみ行い、連動撃破は delta 適用後に
  frameTime > 0 かつ frameTime < 6000 であることだけチェック」とする方法もある。
  
  完全なバリデーションが必要なら、仮の defeats 配列を作って calculateSpawns を再実行し、
  全撃破が有効かを検証する。ドラッグ中の毎フレーム呼び出しにはコストが高いが、
  撃破数が少ないため実用上は問題ない。

### `src/hooks/scenarioReducer.ts`

**MOVE_DEFEATS_BATCH アクション追加:**

```ts
| { type: "MOVE_DEFEATS_BATCH"; payload: ReadonlyArray<{ id: string; frameTime: FrameTime }> }
```

### 操作説明カード（Timeline/index.tsx）

凡例の上の操作方法に1行追加:

```
Shift+ドラッグ: 後続撃破も連動移動
```

## 実装の優先度

1. **必須**: useTimelineDrag での連動モード判定・delta計算・確定処理
2. **必須**: MOVE_DEFEATS_BATCH reducer アクション
3. **必須**: 操作説明カードの更新
4. **推奨**: ドラッグ中の連動撃破プレビュー表示
5. **任意**: Shift ホバー時の連動対象ハイライト

4, 5 は実装が複雑になるなら省略してよい。まずは 1-3 で動作確認し、余裕があれば追加する。

## 完了条件
- Shift なしドラッグで従来通り個別移動すること
- Shift ありドラッグで同枠の後続撃破が連動して移動すること
- 連動移動でバリデーション不正になる場合、移動が拒否されること（ドラッグ中に赤表示等）
- 連動移動確定後、湧きが正しく再計算されること
- 操作説明カードに「Shift+ドラッグ: 後続撃破も連動移動」が表示されること
- ESCキーで連動ドラッグをキャンセルできること
