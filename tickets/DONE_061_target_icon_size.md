# 061: タイムラインのターゲットアイコンを2倍のサイズにする

## 種別
UI改善

## 問題
湧きマーカー右のラベルに表示されるターゲットブキアイコンが 14x14 で小さすぎて判別しにくい。

## 修正箇所
- `src/components/Timeline/SpawnMarker.tsx`

## 修正内容

アイコンサイズを 14x14 → 28x28 に変更:

```tsx
// 変更前
style={{ width: 14, height: 14, marginLeft: 2, verticalAlign: "middle" }}

// 変更後
style={{ width: 28, height: 28, marginLeft: 2, verticalAlign: "middle" }}
```

## 完了条件
- ターゲットブキアイコンが約28x28pxで表示されること
- ラベル全体のレイアウトが崩れないこと
