# 027: 方面選択フロートをラベル上に直接表示し、マウス移動中に消えないようにする

## 種別
UI改善

## 問題
現在の方面選択フロートは方面ラベルの右側に表示されるが、ラベルからフロートへマウスを移動する途中にラベルの `onMouseLeave` が発火してフロートが消えてしまう。

## 修正箇所
- `src/components/Timeline/DirectionLabels.tsx` — `DirectionLabel` コンポーネント

## 修正内容

### フロートの表示位置を変更
フロートを方面ラベルの右側ではなく、**方面ラベルの上に重ねて表示する**。「正面」という文字がある場所にそのままフロート（選択ボタン群）が出現するイメージ。

```tsx
{/* ホバー時フロート — ラベルの中央に重ねて表示 */}
{hovered && (
  <div
    className="absolute flex items-center gap-1 rounded border border-border bg-surface p-1 shadow-sm"
    style={{
      left: '50%',
      top: '50%',
      transform: 'translate(-50%, -50%)',
      zIndex: 20,
      whiteSpace: 'nowrap',
    }}
  >
    {PRESET_NAMES.map((preset) => (
      <button ...>{preset}</button>
    ))}
  </div>
)}
```

### ホバー判定の範囲
フロートが方面ラベルの上に重なるため、`onMouseEnter` / `onMouseLeave` のハンドラはラベルの親 div に設定されたままでよい。フロート自体もこの親 div の内部にあるので、フロートにマウスが入っても `onMouseLeave` は発火しない。

ただし、フロートが親 div の範囲を超える（横幅が広い）場合は `onMouseLeave` が発火しうるので、親 div に `overflow: visible` を設定し、さらに onMouseLeave のハンドラで relatedTarget がフロート内かどうかをチェックする:

```tsx
const containerRef = useRef<HTMLDivElement>(null);

const handleMouseLeave = useCallback((e: React.MouseEvent) => {
  // マウスがフロート内の要素に移動した場合は閉じない
  const related = e.relatedTarget as Node | null;
  if (containerRef.current?.contains(related)) return;
  setHovered(false);
}, []);
```

## 完了条件
- 方面ラベルにホバーすると、ラベルの上に選択ボタン群が表示されること
- フロート上にマウスを移動しても消えないこと
- フロート外にマウスを出すと消えること
- 選択ボタンをクリックすると方面が確定してフロートが閉じること
