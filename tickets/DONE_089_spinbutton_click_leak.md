# 089: 秒数入力のスピンボタン（▲▼）クリック時に❌が出る問題の修正

## 種別
バグ

## 問題

DefeatMarker や ElapsedTimeLabel の `<input type="number">` のスピンボタン（▲▼）をクリックすると、クリックイベントが GrillSlotLane の `handleClick`（撃破追加）まで伝搬し、バリデーション失敗の❌マークが表示される。

## 原因

`<input type="number">` のスピンボタンをクリックすると:
1. `onChange` で値が変更される
2. `click` イベントが input → 親要素 → ... → GrillSlotLane まで伝搬

DefeatMarker は `handleClick` で `e.stopPropagation()` をしているが、これは editing が false の時のマーカー本体のクリックに対する処理。editing 中の input のスピンボタンクリックは、input の `onMouseDown` で `stopPropagation` しているものの、`click` イベントの伝搬は別途止める必要がある。

## 修正箇所

### DefeatMarker.tsx

editing 中の input を囲むコンテナ（または input 自体）で `onClick` の伝搬も止める:

```tsx
{editing ? (
  <input
    ref={inputRef}
    type="number"
    ...
    onMouseDown={handleInputMouseDown}
    onClick={(e) => e.stopPropagation()}  // 追加
  />
) : (
  ...
)}
```

### ElapsedTimeLabel.tsx

同様に input に `onClick` の伝搬停止を追加:

```tsx
<input
  ref={inputRef}
  type="number"
  ...
  onMouseDown={handleInputMouseDown}
  onClick={(e) => e.stopPropagation()}  // 追加
/>
```

## 完了条件
- DefeatMarker の秒数入力のスピンボタン（▲▼）をクリックしても❌が出ないこと
- ElapsedTimeLabel の秒数入力のスピンボタン（▲▼）をクリックしても❌が出ないこと
- 値が正常に増減すること
- 入力欄外のタイムラインクリックは従来通り撃破追加が動作すること
