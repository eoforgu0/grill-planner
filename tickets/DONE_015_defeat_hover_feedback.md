# 015: 撃破マーカーにホバー状態の視覚フィードバックを追加

## 種別
UI改善

## 問題
設計書 06_UI_DESIGN §4.2 で撃破マーカーのホバー時に `--color-defeat-hover` + `cursor: grab` と定義されているが、現在は cursor: grab のみでホバー時の色変更がない。ドラッグ可能であることがユーザーに伝わりにくい。

## 修正箇所
- `src/components/Timeline/DefeatMarker.tsx`

## 修正内容
マーカーのダイヤモンド div にホバー時の色変更を追加:

```tsx
// ドラッグ中でない場合のみホバー色を適用
const isHoverable = !isDragging;
```

CSS で対応する場合は、インラインスタイルではなく className ベースで:
```css
.defeat-marker:hover {
  background-color: var(--color-defeat-hover);
}
```

またはインラインスタイルで onMouseEnter/onMouseLeave を使用。ただし transition が設計書 §9 で定義されているので `transition: background-color 0.15s` も合わせて適用すること（既に実装済みの可能性あり）。

## 完了条件
- 撃破マーカーにマウスホバーすると色が `--color-defeat-hover` に変化すること
- ドラッグ中はホバー色ではなくドラッグ色が優先されること
- 色変化に 150ms 程度のトランジションがあること
