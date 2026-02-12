# 057: 活動期間バー（ActivePeriod）をマーカーと同じ水平位置に配置する

## 種別
UI改善

## 問題
A枠・B枠それぞれの右側にある活動期間バー（ActivePeriod）が、湧きマーカーや撃破マーカーとは異なる水平位置（`right: 4` でレーン右端寄せ）に配置されている。マーカーは `left: 50%` でレーン中央に配置されるため、バーとマーカーが同一直線上にない。

## 修正箇所
- `src/components/Timeline/ActivePeriod.tsx` — 水平位置の変更

## 修正内容
ActivePeriod の水平位置をマーカーと同じ「レーン中央」に変更する:

```tsx
// 変更前
<div
  className="absolute"
  style={{
    top: topY,
    right: 4,
    height,
    width: ACTIVITY_BAR_WIDTH,
    ...
  }}
/>

// 変更後
<div
  className="absolute"
  style={{
    top: topY,
    left: '50%',
    transform: 'translateX(-50%)',
    height,
    width: ACTIVITY_BAR_WIDTH,
    ...
  }}
/>
```

これにより活動期間バーがレーン中央に配置され、湧きマーカー（円）→ 活動期間バー → 撃破マーカー（菱形）が垂直方向の同一直線上に並ぶ。

### z-index の確認
活動期間バーはマーカーの背面に表示されるべきなので `zIndex: 1` のまま維持。マーカーの z-index（2〜4）より低いため問題なし。

## 完了条件
- 活動期間バーがレーン中央に配置され、マーカーと同一の垂直線上にあること
- マーカー（湧き・撃破）が活動期間バーの上に描画されること
- A枠・B枠ともに正しく配置されていること
