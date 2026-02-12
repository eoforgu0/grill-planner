# 068: B枠グレーアウトのz-indexを下げる

## 種別
バグ（表示）

## 問題
B枠のグリル未出現時間の灰色エリア（`zIndex: 5`）が、湧きマーカー（`zIndex: 3`）やフキダシ、ターゲットアイコンの上に被さっている。

## 修正箇所
- `src/components/Timeline/GrillSlotLane.tsx`

## 修正内容

灰色エリアの zIndex を 5 → 2 に下げる:

```tsx
{/* B枠の無効エリア（自動湧きより前） */}
{inactiveAboveFrame != null && (
  <div
    className="pointer-events-none absolute inset-x-0 top-0"
    style={{
      height: frameToPixelY(inactiveAboveFrame),
      backgroundColor: "rgba(128, 128, 128, 0.3)",
      zIndex: 2,  // was 5 → マーカー(3)やラベルの下に
    }}
  />
)}
```

現在の z-index 一覧:
- ActivePeriod: 1
- DirectionBands: 0（背景）
- SpawnMarker: 3
- DefeatMarker: 4（ドラッグ中は6）
- バリデーション失敗フィードバック: 10

灰色エリアは 2 にすることで、ActivePeriod（1）の上・マーカー（3）の下に位置する。

## 完了条件
- 灰色エリアの上にある湧きマーカー、フキダシ、ターゲットアイコンが灰色に隠されないこと
- 灰色エリアが方面バンドや活動期間バーの上には描画されること
