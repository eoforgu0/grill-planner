# 050: 表示・ターゲット基準・方面選択フロートのボタンを Button Group 化する

## 種別
UI改善

## 問題
現在の択一選択ボタンが独立したボタンとして並んでいるため、択一選択であることが視覚的にわかりにくい。Button Group（ボタンが連結した1つのコントロールとして見える）にする。

## 対象箇所
1. 表示モード切替（DisplayModeToggle）: アイコン / テキスト / 両方
2. ターゲット基準切替（ScenarioView 内）: ブキ / プレイヤー
3. 方面選択フロート（DirectionLabels 内）: プリセット名3つ

## 修正箇所
- `src/components/Settings/DisplayModeToggle.tsx`
- `src/ScenarioView.tsx` — ターゲット基準ボタン
- `src/components/Timeline/DirectionLabels.tsx` — フロート内ボタン

## 修正内容

### Button Group スタイル

隣接するボタンの角丸を打ち消して連結した見た目にする:

```tsx
<div className="inline-flex rounded-md border border-border overflow-hidden">
  {options.map((option, i) => (
    <button
      key={option.value}
      onClick={() => onChange(option.value)}
      className={[
        'px-2 py-0.5 text-xs',
        // 隣接ボタンの左境界線
        i > 0 ? 'border-l border-border' : '',
        // 選択状態
        selected === option.value
          ? 'bg-primary text-white'
          : 'bg-surface text-text hover:bg-bg',
      ].join(' ')}
    >
      {option.label}
    </button>
  ))}
</div>
```

ポイント:
- 外側の div に `rounded-md border overflow-hidden` で角丸の枠を作る
- 各ボタンには個別の角丸をつけない
- 隣接ボタン間は `border-l` で区切り線
- 選択状態は `bg-primary text-white`、非選択は `bg-surface text-text hover:bg-bg`

### 共通コンポーネント化（推奨）

3箇所で同じパターンを使うため、共通の ButtonGroup コンポーネントを作成してもよい:

```tsx
// src/components/ButtonGroup.tsx
interface ButtonGroupProps<T extends string> {
  options: readonly { value: T; label: string }[];
  selected: T;
  onChange: (value: T) => void;
  size?: 'xs' | 'sm';
}
```

### 方面選択フロートの Button Group

フロート内のプリセットボタンも同じ Button Group スタイルにする。フロートの `p-1 gap-1` を `overflow-hidden rounded-md border` に変更。

## 完了条件
- 表示モード切替がボタンが連結した1つのグループとして表示されること
- ターゲット基準切替が同様のグループ表示であること
- 方面選択フロートのボタンが同様のグループ表示であること
- 選択状態が明確に区別できること
- ホバー時のフィードバックがあること
