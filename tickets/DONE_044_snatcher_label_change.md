# 044: 「タマヒロイメモ」→「タマヒロイ方向」にラベル変更、プレースホルダを「右ヒロイ」に変更

## 種別
UI改善

## 修正箇所
- `src/components/Settings/MemoSection.tsx`

## 修正内容

```tsx
// 変更前
<label className="mb-1 block text-xs text-text-muted">タマヒロイメモ</label>
<input ... placeholder="自由メモ" ... />

// 変更後
<label className="mb-1 block text-xs text-text-muted">タマヒロイ方向</label>
<input ... placeholder="右ヒロイ" ... />
```

## 完了条件
- ラベルが「タマヒロイ方向」に変更されていること
- プレースホルダが「右ヒロイ」に変更されていること
