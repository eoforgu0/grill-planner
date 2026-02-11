# 037: 方面選択フロートが左側に画面外へはみ出る問題を修正

## 種別
バグ

## 問題
方面選択のフロート（「左」「正面」「右」ボタン群）が `left: 50%; transform: translate(-50%, -50%)` で方面ラベルの中央に配置されるが、方面ラベル列は画面左端付近にあるため、フロートの左半分が画面外に飛び出して押せない。

## 修正箇所
- `src/components/Timeline/DirectionLabels.tsx`

## 修正内容
フロートの配置を「方面ラベルの中央」ではなく「方面ラベルの左端を基準に右方向へ展開」に変更する。

```tsx
{/* ホバー時フロート — ラベル左端から右方向に展開 */}
{hovered && (
  <div
    className="absolute flex items-center gap-1 rounded border border-border bg-surface p-1 shadow-sm"
    style={{
      left: 0,
      top: '50%',
      transform: 'translateY(-50%)',
      zIndex: 20,
      whiteSpace: 'nowrap',
    }}
  >
    ...
  </div>
)}
```

変更点: `left: '50%'` → `left: 0`, `transform: 'translate(-50%, -50%)'` → `transform: 'translateY(-50%)'`

これによりフロートは方面ラベルの左端から右方向に伸び、レーン上に重なる形になるが画面外には出ない。

## 完了条件
- フロートの全ボタンが画面内に表示され、クリック可能であること
- フロートが方面ラベルと縦方向で中央揃えされていること
