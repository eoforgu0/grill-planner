# 077: 凡例にスポナー確定マーカーと湧き抑制の説明を追加

## 種別
UI改善

## 修正箇所
- `src/components/Timeline/index.tsx` — 凡例セクション

## 修正内容

既存の凡例（湧き、撃破）の下に2項目追加:

```tsx
{/* 凡例 */}
<div className="mt-2 border-t border-border pt-2">
  <div className="mb-1 text-xs font-medium">凡例</div>

  {/* 既存: 湧き */}
  <div className="flex items-center gap-1.5">
    <div style={{
      width: 10, height: 10, borderRadius: "50%",
      backgroundColor: "var(--color-spawn)",
      border: "1.5px solid var(--color-slot-a)",
      flexShrink: 0,
    }} />
    <span>湧き</span>
  </div>

  {/* 既存: 撃破 */}
  <div className="flex items-center gap-1.5">
    <div style={{
      width: 10, height: 10,
      backgroundColor: "var(--color-defeat)",
      transform: "rotate(45deg)",
      flexShrink: 0,
    }} />
    <span>撃破</span>
  </div>

  {/* 追加: スポナー確定 */}
  <div className="flex items-center gap-1.5">
    <div style={{
      width: 10, height: 10,
      display: "flex", alignItems: "center", justifyContent: "center",
      flexShrink: 0,
    }}>
      <div style={{
        width: 7, height: 7, borderRadius: "50%",
        backgroundColor: "var(--color-spawner-decision)",
        border: "1px solid var(--color-respawn-line)",
      }} />
    </div>
    <span>スポナー確定</span>
  </div>

  {/* 追加: 湧き（抑制） */}
  <div className="flex items-center gap-1.5">
    <div style={{
      width: 10, height: 10, borderRadius: "50%",
      backgroundColor: "var(--color-spawn)",
      border: "1.5px dashed var(--color-slot-a)",
      flexShrink: 0,
    }} />
    <span>湧き（抑制）</span>
  </div>
</div>
```

スポナー確定マーカーは実際の RespawnConnector と同じ見た目（黄色●＋灰色ボーダー）。
湧き抑制マーカーは通常の湧きと同じだがボーダーが破線（dashed）。

## 完了条件
- 凡例に「スポナー確定」（黄色小丸）が表示されること
- 凡例に「湧き（抑制）」（破線ボーダー丸）が表示されること
- 既存の「湧き」「撃破」の凡例はそのまま維持されること
