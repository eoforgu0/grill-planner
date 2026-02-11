# 029: スポナー確定マーカーを撃破-湧き間に追加

## 種別
UI改善

## 問題
現在、撃破→湧きの間にはリスポーン接続線上にスポナー決定マーク（小◇）が描画されているが、サイズが小さく目立たない。湧きマーカー（●）よりもさらに小さい丸マーカーとしてスポナー確定位置を明示したい。

## 修正箇所
- `src/components/Timeline/RespawnConnector.tsx`

## 修正内容
現在の polygon（◇）を、湧きマーカーよりも小さい丸（●）に変更する:

- 形状: 円（●）
- サイズ: 湧きマーカーの半分程度（例: MARKER_SIZE の 50%、7〜8px）
- 色: `--color-spawner-decision`（黄色系）
- 枠線: 1px solid `--color-respawn-line`

```tsx
<circle
  cx={x}
  cy={decisionY}
  r={MARKER_SIZE * 0.25}
  fill="var(--color-spawner-decision)"
  stroke="var(--color-respawn-line)"
  strokeWidth={1}
/>
```

## 完了条件
- 撃破マーカーと湧きマーカーの間にスポナー確定マーカー（小丸）が表示されること
- サイズが湧きマーカーよりも明確に小さいこと
