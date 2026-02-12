# 053: 方面別統計テーブルの文字サイズを縮小する

## 種別
UI改善

## 問題
方面別統計テーブルの「方面」「湧き」「撃破」列の文字サイズが `text-sm`（14px）で、右ペインのコンパクトなUIに対して大きい。# 列の `text-xs` に合わせて縮小する。

## 修正箇所
- `src/components/Statistics/DirectionStatsTable.tsx`

## 修正内容
テーブル本体の `text-sm` を `text-xs`（12px）に変更:

```tsx
// 変更前
<table className="w-full text-sm">

// 変更後
<table className="w-full text-xs">
```

合計行の font-bold はそのまま維持。

方面名別合計内の表も同様に `text-xs` に統一:
```tsx
// 変更前
<table className="w-full text-sm">

// 変更後
<table className="w-full text-xs">
```

## 完了条件
- 方面別統計テーブルの方面・湧き・撃破列が text-xs で表示されること
- 方面名別合計のテーブルも同様に text-xs で表示されること
- 合計行は引き続き太字であること
