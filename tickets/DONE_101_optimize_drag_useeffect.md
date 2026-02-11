# 101: useTimelineDrag の useEffect 依存配列を最適化

## 種別
改善（パフォーマンス）

## 問題
`src/hooks/useTimelineDrag.ts` の useEffect 依存配列に `dragState.isDragging`, `dragState.dragFrameTime`, `dragState.isValidPosition` が含まれている。ドラッグ中は mousemove のたびにこれらが変化するため、毎フレーム window イベントリスナーの removeEventListener → addEventListener が発生する。

## 修正箇所
- `src/hooks/useTimelineDrag.ts`

## 修正内容
dragState とコールバック関数を useRef に格納し、useEffect の依存配列からドラッグ中に変化するフィールドを除外する。

```ts
const dragStateRef = useRef(dragState);
dragStateRef.current = dragState;

const onDragEndRef = useRef(onDragEnd);
onDragEndRef.current = onDragEnd;

// 他のコールバックも同様に useRef 化

useEffect(() => {
  const handleMouseMove = (e: MouseEvent) => {
    // dragStateRef.current を参照
  };
  const handleMouseUp = () => {
    // dragStateRef.current, onDragEndRef.current を参照
  };
  // ...
  window.addEventListener('mousemove', handleMouseMove);
  window.addEventListener('mouseup', handleMouseUp);
  window.addEventListener('keydown', handleKeyDown);
  return () => { /* cleanup */ };
}, []); // 依存配列を空にする
```

## 完了条件
- useEffect の依存配列に dragState 関連のフィールドが含まれていないこと
- ドラッグ操作が従来通り動作すること（Playwright で確認推奨）
