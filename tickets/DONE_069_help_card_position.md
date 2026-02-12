# 069: 操作説明カードの位置をB枠右端+20pxにする

## 種別
UI改善

## 問題
操作説明カードが右すぎる。B枠の右端から20px右の位置にカードの左端を合わせたい。

## 修正箇所
- `src/components/Timeline/index.tsx` — 操作説明カードの位置指定

## 修正内容

現在 `right: -280` のような絶対値で配置しているが、これだとレーン幅変更のたびに調整が必要。`left` ベースの配置に変更し、レーン領域の幅（`lanesWidth`）に対して正確に位置づける:

```tsx
{/* 操作説明（レーン右側） */}
<div
  className="pointer-events-none absolute select-none"
  style={{
    left: lanesWidth + 20,  // レーン領域の右端 + 20px
    top: 8,
    zIndex: 0,
  }}
>
  ...
</div>
```

`lanesWidth` は `showBSlot ? LANE_WIDTH * 2 + LANE_SPACING : LANE_WIDTH` で計算済み。これを使うことでレーン幅やB枠の有無に自動追従する。

## 完了条件
- 操作説明カードの左端がB枠の右端から約20pxの位置にあること
- B枠が非表示（キケン度が低い場合）でもA枠の右端+20pxの位置に正しく配置されること
