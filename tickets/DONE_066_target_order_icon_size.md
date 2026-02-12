# 066: ターゲット順テーブルのブキアイコンを少し大きくする

## 種別
UI改善

## 問題
ターゲット順の各行のButtonGroup内のブキアイコンが `h-4 w-4`（16x16px）で小さすぎて判別しにくい。

## 修正箇所
- `src/components/ButtonGroup.tsx` — アイコンサイズ（または TargetOrderTable 専用のサイズ指定）

## 修正内容

ButtonGroup のアイコンサイズを `h-4 w-4` → `h-5 w-5`（20x20px）に変更:

```tsx
// 変更前
<div className="ml-0.5 h-4 w-4 shrink-0">
  {option.icon && <img src={option.icon} alt="" className="h-4 w-4" />}
</div>

// 変更後
<div className="ml-0.5 h-5 w-5 shrink-0">
  {option.icon && <img src={option.icon} alt="" className="h-5 w-5" />}
</div>
```

他の ButtonGroup（表示モード、方面選択フロート等）はアイコンを使用していないため影響なし。

## 完了条件
- ターゲット順のブキアイコンが 20x20px で表示されること
- ButtonGroup のレイアウトが崩れないこと
