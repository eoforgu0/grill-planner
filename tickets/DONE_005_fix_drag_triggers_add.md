# 005: ドラッグ未遂時にクリックイベントが発火しマーカーが追加される

## 種別
バグ（致命的）

## 症状
撃破マーカーをドラッグしようとすると、mouseup 時にレーンの onClick が発火し、新しい撃破マーカーが追加される。

## 原因分析
イベントフローを時系列で追うと以下の問題がある:

### ケース: ドラッグ閾値（5px）未満でマウスを離した場合

1. DefeatMarker の `onMouseDown` → `e.stopPropagation()` で **mousedown のレーン伝播は止まる**
2. `startDragCandidate()` → `candidateRef.current` にセット
3. マウスを少し動かす（< 5px）→ threshold 未満なのでドラッグ開始しない
4. mouseup → `handleMouseUp` で `candidateRef` をクリア、`dragState` をリセット
5. **ブラウザの click イベント**: mousedown + mouseup = click。DefeatMarker の mousedown で `stopPropagation` しているが、**click イベントは mousedown/mouseup とは別のイベント**。DefeatMarker の div 上で mousedown → mouseup が起きると、**click イベントが DefeatMarker の div で発火し、バブリングでレーン div の onClick に到達する**。

DefeatMarker には `onClick` ハンドラも `e.stopPropagation()` もないため、click バブリングが止まらずレーンの `handleClick` が呼ばれる。

### ケース: ドラッグ閾値を超えた場合

1. ドラッグ中、`dragState.isDragging = true`
2. mouseup → `handleMouseUp` で onDragEnd を呼び移動を確定、`setDragState(INITIAL_STATE)` でリセット
3. **直後に click イベントが発火** → `handleClick` で `dragState.isDragging` をチェック
4. しかし `setDragState(INITIAL_STATE)` は非同期（React の state 更新）なので、click ハンドラ実行時にはまだ `isDragging = true` かどうかが **不確定**。
5. React 18+ の batching によりほぼ同じタイミングで更新されるが、ブラウザの click イベントは同期的に発火するため、`dragState.isDragging` のチェックが **レースコンディション** になる。

さらに根本的な問題として:

6. `handleClick` の依存配列に `dragState.isDragging` が含まれている:
   ```ts
   [slot, onAddDefeat, dragState.isDragging]
   ```
   しかし useTimelineDrag の useEffect は `[laneRef]` のみに依存しているため、dragState の更新と handleClick の更新タイミングにずれが生じうる。

## 修正箇所
- `src/hooks/useTimelineDrag.ts`
- `src/components/Timeline/GrillSlotLane.tsx`

## 修正内容

### 修正A: ドラッグ終了直後の click 抑止フラグを追加

useTimelineDrag に「直前にドラッグ操作があったか」を示すフラグを追加し、GrillSlotLane の click ハンドラで参照する:

```ts
// useTimelineDrag.ts
const justFinishedDragRef = useRef(false);

// handleMouseUp 内:
justFinishedDragRef.current = true;
requestAnimationFrame(() => {
  justFinishedDragRef.current = false;
});

// 返値に追加:
return { dragState, startDragCandidate, cancelDrag, justFinishedDragRef };
```

```ts
// GrillSlotLane.tsx の handleClick 内:
if (justFinishedDragRef.current) return;
```

### 修正B: DefeatMarker で click イベントの伝播も止める

```tsx
// DefeatMarker.tsx に追加:
const handleClick = useCallback((e: MouseEvent) => {
  e.stopPropagation();
}, []);

// JSX:
<div ... onMouseDown={handleMouseDown} onClick={handleClick} onContextMenu={handleContextMenu}>
```

**両方の修正を適用すること**。修正Aだけだとドラッグしていない単純クリック（閾値未満）でも問題が残り、修正Bだけだとドラッグ完了後の click 発火位置がマーカー外だった場合に効かない。

### 修正C: handleClick の dragState.isDragging チェックを ref ベースに変更

`dragState.isDragging` は React state なので非同期。candidateRef の存在チェックの方が確実:

```ts
// GrillSlotLane.tsx の handleClick 内で、isDragging の代わりに:
if (justFinishedDragRef.current) return;
// （修正Aと同じ）
```

そして `handleClick` の依存配列から `dragState.isDragging` を削除する。

## 完了条件
- 撃破マーカーを左クリック（mousedown → mouseup、移動なし）しても新しいマーカーが追加されないこと
- 撃破マーカーをドラッグして離しても新しいマーカーが追加されないこと
- レーンの空白部分をクリックしたときは従来通りマーカーが追加されること
- Playwright で確認推奨
