# 056: A枠・B枠のラベルテキストを削除する

## 種別
UI改善

## 問題
「A枠」「B枠」というラベルテキストがレーン上部に表示されているが、プレイヤー間でこの表現は一般的でなく混乱を招く。

## 修正箇所
- `src/components/Timeline/GrillSlotLane.tsx`

## 修正内容
枠ラベルの表示部分を削除:

```tsx
// 削除
<span
  className="absolute select-none text-sm font-bold"
  style={{ left: 4, top: 6, color: slotColor, zIndex: 5 }}
>
  {slot}枠
</span>
```

レーンの区別は上端の色付きボーダー（`borderTop: 3px solid ${slotColor}`）で視覚的に十分判別可能。

## 注意
開発上はA枠・B枠の概念を引き続き使用する。コード内の変数名やコメントは変更しない。あくまでUI上の表示テキストのみ削除。

## 完了条件
- レーン上に「A枠」「B枠」のテキストが表示されないこと
- レーン自体の見た目（色付きボーダー等）は維持されていること
