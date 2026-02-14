# 078: 方面選択フロートがマウス高速離脱時に残る問題の修正

## 種別
バグ（操作）

## 問題

方面ラベルにホバーして方面選択のButtonGroupフロートが表示された状態で、マウスが高速にウィンドウ外やタイムラインエリアに移動すると、`mouseleave` イベントがブラウザから発火されず、フロートが表示されたままになる。

再度方面ラベル領域にマウスを当てるまで消えない。

## 原因

ブラウザの仕様として、マウスが高速移動した場合に `mouseleave` イベントが発火されないことがある。現在の実装は `onMouseLeave` のみに依存しているため、このケースに対処できない。

## 修正箇所
- `src/components/Timeline/DirectionLabels.tsx` — `DirectionLabel` コンポーネント

## 修正内容

`useEffect` で `document` レベルの `mousemove` リスナーを登録し、マウスが要素外にあることを検出して閉じる:

```tsx
function DirectionLabel({ ... }: DirectionLabelProps) {
  const [hovered, setHovered] = useState(false);
  const containerRef = useRef<HTMLDivElement>(null);

  // フロート表示中のみ document.mousemove を監視して要素外を検出
  useEffect(() => {
    if (!hovered) return;

    const handleDocumentMouseMove = (e: globalThis.MouseEvent) => {
      if (!containerRef.current) return;
      const rect = containerRef.current.getBoundingClientRect();
      const { clientX, clientY } = e;

      // フロートはleft:0 + ButtonGroupの幅分はみ出すため、
      // 左方向のマージンを広めに取る（ButtonGroupの幅を概算）
      const margin = 8; // 上下左右のマージン
      const isOutside =
        clientX < rect.left - margin ||
        clientX > rect.right + margin ||
        clientY < rect.top - margin ||
        clientY > rect.bottom + margin;

      if (isOutside) {
        setHovered(false);
      }
    };

    document.addEventListener("mousemove", handleDocumentMouseMove);
    return () => {
      document.removeEventListener("mousemove", handleDocumentMouseMove);
    };
  }, [hovered]);

  // 既存の handleMouseLeave, handleSelect 等はそのまま維持
  // ...
}
```

### なぜこの方法が有効か

- `mouseleave` が発火されなくても、`document.mousemove` は必ず発火される（マウスがブラウザ内にある限り）
- マウスがウィンドウ外に出た場合はイベントが来なくなるが、ウィンドウに戻った瞬間に次の `mousemove` で判定される
- `hovered === false` の時はリスナーが登録されないため、パフォーマンスへの影響は最小限

### マージンの考慮

ButtonGroupフロートは `containerRef` の要素上に absolute で配置されており、`containerRef.current.getBoundingClientRect()` にはフロート部分が含まれない場合がある。そのため `margin` で少し余裕を持たせる。

ただし厳密にフロートの矩形を含めたい場合は、フロート用の ref を追加して両方の矩形でチェックする:

```tsx
const floatRef = useRef<HTMLDivElement>(null);

// mousemove 内で
const isInsideContainer = isPointInRect(clientX, clientY, containerRef, margin);
const isInsideFloat = floatRef.current && isPointInRect(clientX, clientY, floatRef, margin);
if (!isInsideContainer && !isInsideFloat) {
  setHovered(false);
}
```

→ こちらの方が堅牢。推奨。

## 完了条件
- 方面ラベルにホバーしてフロートが表示された状態で、マウスを高速にウィンドウ外に移動させてもフロートが消えること（ブラウザに戻った時点で消える）
- 方面ラベルにホバーしてフロートが表示された状態で、マウスをタイムラインエリアに移動させるとフロートが消えること
- フロート上にマウスがある状態ではフロートが維持されること
- フロート上のボタンをクリックして方面を変更できること（既存動作を維持）
- 通常のホバー→フロート表示→離脱の動作に影響がないこと
