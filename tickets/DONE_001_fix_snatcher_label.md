# 001: MemoSection のラベル「ハコビヤ」→「タマヒロイ」

## 種別
バグ（用語不一致）

## 問題
`src/components/Settings/MemoSection.tsx` 内のラベルが「ハコビヤメモ」になっているが、設計書（01_PROJECT_OVERVIEW 用語集、03_REQUIREMENTS FR-01-6）では「タマヒロイメモ」と定義されている。ハコビヤ（ステージにインクを落とすNPC）とタマヒロイ（金イクラを自動回収するNPC = Snatcher）は別の存在。

## 修正箇所
- `src/components/Settings/MemoSection.tsx`

## 修正内容
```
ハコビヤメモ → タマヒロイメモ
```

## 完了条件
- ラベルが「タマヒロイメモ」に変更されていること
