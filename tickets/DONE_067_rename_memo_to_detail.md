# 067: 「メモ」→「詳細」にリネームし、開閉ボタンを細くし、方面名設定を詳細パネル内に移動する

## 種別
UI改善

## 問題
1. ブキ/SP選択はターゲット表示にも関係しており、もはやメモではない。「詳細」に改名する。
2. メモの開閉ボタンの縦幅が `py-2`（上下8px）で太すぎる。
3. 方面名の設定が設定パネル上部にあるが、詳細パネル内に移動して整理する。

## 修正箇所
- `src/components/Settings/MemoSection.tsx` — ラベル変更 + パディング変更 + 方面名入力の追加
- `src/ScenarioView.tsx` — 方面名設定を設定パネル上部から削除、MemoSection に props 追加

## 修正内容

### 1. 「メモ」→「詳細」

```tsx
// 変更前
<span>メモ</span>

// 変更後
<span>詳細</span>
```

### 2. 開閉ボタンの縦幅を縮小

```tsx
// 変更前
className="flex w-full items-center gap-2 px-4 py-2 text-sm ..."

// 変更後
className="flex w-full items-center gap-2 px-4 py-1 text-sm ..."
```

`py-2` → `py-1` で上下4pxに縮小。

### 3. 方面名設定を詳細パネル内に移動

ScenarioView の設定パネル上部から方面名の以下を削除:
```tsx
// ScenarioView から削除
<div className="flex items-center gap-2">
  <span className="text-sm text-text-muted">方面名:</span>
  {state.directionPresets.map(...)}
</div>
```

MemoSection に props を追加:
```tsx
interface MemoSectionProps {
  // ... 既存 props
  directionPresets: readonly [string, string, string];
  onSetDirectionPreset: (index: 0 | 1 | 2, name: string) => void;
}
```

MemoSection 内の配置: シナリオコード + タマヒロイ方向の行の下（またはタマヒロイ方向の右）に方面名入力を配置:

```tsx
{/* シナリオコード + タマヒロイ方向 + 方面名（横並び） */}
<div className="flex items-end gap-4">
  <div>
    <label ...>シナリオコード</label>
    <input ... />
  </div>
  <div>
    <label ...>タマヒロイ方向</label>
    <input ... />
  </div>
  <div>
    <label className="mb-1 block text-xs text-text-muted">方面名</label>
    <div className="flex gap-1">
      {directionPresets.map((preset, i) => (
        <input
          key={i}
          type="text"
          value={preset}
          onChange={(e) => onSetDirectionPreset(i as 0 | 1 | 2, e.target.value)}
          className="w-16 rounded-sm border border-border bg-surface px-1 py-0.5 text-center text-xs text-text"
          maxLength={10}
        />
      ))}
    </div>
  </div>
</div>
```

## 完了条件
- 開閉ボタンのラベルが「詳細」であること
- 開閉ボタンの縦幅が元より細くなっていること
- 方面名の入力欄が詳細パネル内（タマヒロイ方向の右）に配置されていること
- 設定パネル上部から方面名の入力欄が削除されていること
