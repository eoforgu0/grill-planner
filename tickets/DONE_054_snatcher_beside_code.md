# 054: タマヒロイ方向の入力欄をシナリオコードの右に移動する

## 種別
UI改善

## 問題
タマヒロイ方向の入力欄がシナリオコードとは別の行にあり、レイアウト上の無駄な空間が生じている。シナリオコードの右横に配置してスペースを有効活用する。

## 修正箇所
- `src/components/Settings/MemoSection.tsx`

## 修正内容
シナリオコードとタマヒロイ方向を同一行に横並びで配置する:

```tsx
{/* シナリオコード + タマヒロイ方向（横並び） */}
<div className="flex items-end gap-4">
  <div>
    <label className="mb-1 block text-xs text-text-muted">シナリオコード</label>
    <input ... />
  </div>
  <div>
    <label className="mb-1 block text-xs text-text-muted">タマヒロイ方向</label>
    <input ... />
  </div>
</div>
```

元の「サッチャーメモ」の独立した div ブロックは削除する。

## 完了条件
- シナリオコードとタマヒロイ方向が同一行に横並びで表示されること
- 各入力欄のラベルが正しく表示されること
- レイアウトが崩れないこと
