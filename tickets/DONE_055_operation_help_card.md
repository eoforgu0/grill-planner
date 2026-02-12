# 055: 操作説明を枠で囲い、B枠から離す

## 種別
UI改善

## 問題
1. 操作説明（クリック/ドラッグ/右クリック）がただのテキストで、操作説明であることが感覚的にわかりにくい。
2. B枠に近すぎてB枠についてのテキストに見える。

## 修正箇所
- `src/components/Timeline/index.tsx` — 操作説明エリアのスタイル

## 修正内容

### 枠で囲う
操作説明を薄いボーダーの角丸ボックスで囲い、控えめなヘルプカード風にする:

```tsx
<div
  className="pointer-events-none absolute select-none"
  style={{
    right: -140,   // B枠からさらに離す
    top: 8,
    zIndex: 0,
  }}
>
  <div
    className="rounded-md border border-border px-3 py-2"
    style={{
      fontSize: 11,
      color: 'var(--color-text-muted)',
      lineHeight: 1.6,
      backgroundColor: 'var(--color-bg)',
    }}
  >
    <div className="mb-1 text-xs font-medium" style={{ color: 'var(--color-text-muted)' }}>
      操作方法
    </div>
    <div>クリック: 撃破追加</div>
    <div>ドラッグ: 撃破移動</div>
    <div>右クリック: 撃破削除</div>
  </div>
</div>
```

ポイント:
- `border border-border` で薄い線の角丸四角
- `bg-bg`（`#f8fafc`）で背景色をつけてカード感を出す
- 「操作方法」というタイトルを追加
- `right` を `-140` 程度にしてB枠から十分に離す（元は `-110`）
- 世間一般のヘルプカード/チートシートのデザインを参考にした控えめなスタイル

### right の値について
タイムラインの右側の空間に余裕がない場合は、right の値を調整する。操作説明カードが画面外にはみ出ないよう確認すること。

## 完了条件
- 操作説明が薄い枠線の角丸ボックス内に表示されること
- 「操作方法」等のタイトルがあり、操作説明であることがわかること
- B枠レーンから十分な距離があり、B枠の説明には見えないこと
