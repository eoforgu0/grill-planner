# 023: ブキ/SP選択のアイコン幅を予約し、未選択時のレイアウトずれを防止する

## 種別
UI改善

## 問題
ブキ/スペシャル選択で、未選択時にはアイコンが表示されないためアイコン分の幅(24px = h-6 w-6)が確保されず、選択するとアイコンが出現して全体がずれる。

## 修正箇所
- `src/components/Settings/MemoSection.tsx`

## 修正内容
アイコン表示部分を、選択状態に関わらず常に固定サイズ（w-6 h-6）の領域として確保する。

```tsx
{/* アイコン（常にスペース確保） */}
<div className="flex h-6 w-6 shrink-0 items-center justify-center">
  {selectedWeapon && (
    <img src={getWeaponIconPath(selectedWeapon.id)} alt="" className="h-6 w-6" />
  )}
</div>
```

空の div で固定サイズを確保するため、`empty.png` は不要。

## empty.png について
**使用しない。** 空の div による固定幅確保で十分であり、透過画像の読み込みコストを避けられる。削除してよい。

## 完了条件
- 未選択状態でもアイコン分の幅が確保されていること
- ブキ/スペシャルを選択・解除しても隣接セルの位置がずれないこと
- 全4枠すべてで統一的に動作すること
