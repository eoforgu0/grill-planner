# 071: 詳細パネル内の方面名のラベル・入力欄の高さをシナリオコード等と揃える

## 種別
UI改善

## 問題
方面名のラベル位置と入力欄の上端が、シナリオコードやタマヒロイ方向のそれと揃っていない。方面名の入力欄は `w-16 px-1 py-0.5 text-xs` で他の入力欄（`px-2 py-1 text-sm`）より小さく、かつ3つ横並びのためラベルの位置も異なる。

## 修正箇所
- `src/components/Settings/MemoSection.tsx` — 方面名エリアのレイアウト

## 修正内容

現在の構造:
```tsx
<div className="flex items-end gap-4">
  <div>
    <label>シナリオコード</label>  ← mb-1 block text-xs
    <input ... />                  ← px-2 py-1 text-sm
  </div>
  <div>
    <label>タマヒロイ方向</label>
    <input ... />
  </div>
  <div>
    <label>方面名</label>          ← mb-1 block text-xs
    <div className="flex gap-1">   ← 入力欄をラップする div が1段余分
      <input ... />                ← px-1 py-0.5 text-xs （サイズが異なる）
      <input ... />
      <input ... />
    </div>
  </div>
</div>
```

問題点:
1. `items-end` で下揃えしているが、方面名の入力欄は `py-0.5 text-xs` で他より小さいため、ラベルの位置がずれる
2. 方面名の入力欄をラップする `div` が1段余分にある

修正方針: 方面名の入力欄のサイズを他と揃えるか、ラベルの高さを明示的に揃える。

### 方針A（推奨）: 方面名の入力欄のサイズを他に合わせる

方面名の input を他と同じ `py-1 text-sm` にする。幅は `w-16` のままでよい:

```tsx
<div>
  <label className="mb-1 block text-xs text-text-muted">方面名</label>
  <div className="flex gap-1">
    {directionPresets.map((preset, i) => (
      <input
        key={i}
        type="text"
        value={preset}
        onChange={(e) => onSetDirectionPreset(i as 0 | 1 | 2, e.target.value)}
        className="w-16 rounded-sm border border-border bg-surface px-2 py-1 text-center text-sm text-text"
        maxLength={10}
      />
    ))}
  </div>
</div>
```

変更点:
- `px-1 py-0.5 text-xs` → `px-2 py-1 text-sm`（他の入力欄と統一）
- `text-center` は維持

これにより `items-end` で下揃えしたときに全入力欄の上端・下端が自然に揃う。ラベルも同じ `text-xs` + `mb-1` なので高さが一致する。

## 完了条件
- 方面名のラベルがシナリオコード・タマヒロイ方向のラベルと同じ高さに表示されること
- 方面名の入力欄の上端が他の入力欄の上端と揃っていること
- 方面名の入力欄の下端も他の入力欄と揃っていること
