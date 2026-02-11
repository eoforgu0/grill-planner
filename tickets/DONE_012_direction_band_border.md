# 012: 方面帯の区間境界に破線を追加

## 種別
UI改善

## 問題
設計書 06_UI_DESIGN §3.4 に「境界: 区間境界に 1px の破線」と定義されているが未実装。方面帯の色の変わり目だけでは区間の区切りがわかりにくい。

## 修正箇所
- `src/components/Timeline/DirectionBands.tsx`

## 修正内容
各方面帯の上端（境界）に 1px の破線を描画する。最初の帯（100s）は不要。

```tsx
<div
  style={{
    top,
    height,
    backgroundColor: color,
    borderTop: index > 0 ? '1px dashed var(--color-border)' : 'none',
  }}
/>
```

## 完了条件
- 隣接する方面帯の境界に破線が表示されること
- 最初の帯の上端には破線がないこと
