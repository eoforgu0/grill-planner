# 022: ターゲット基準ボタンのホバースタイルを表示モードボタンと統一する + ラベルを「ターゲット基準」に変更

## 種別
UI改善

## 問題
1. 「表示」ボタン群は `hover:bg-bg` でホバー時に背景色が変わるが、「ターゲット」ボタン群にはホバー時のスタイルがない（非選択状態のボタンにホバーフィードバックがない）。
2. ラベルが「ターゲット:」だが「ターゲット基準:」に変更が必要。

## 修正箇所
- `src/ScenarioView.tsx` — ターゲット基準ボタンのクラスとラベル

## 修正内容

### ラベル変更
```tsx
// 変更前
<span className="text-xs text-text-muted">ターゲット:</span>
// 変更後
<span className="text-sm text-text-muted">ターゲット基準:</span>
```

`text-xs` → `text-sm` も `DisplayModeToggle` の `表示:` に合わせること。

### ホバースタイル統一
非選択状態のボタンに `hover:bg-bg` を追加（DisplayModeToggle と同じクラス）:

```tsx
// 変更前
'border border-border bg-surface text-text'
// 変更後
'border border-border bg-surface text-text hover:bg-bg'
```

## 完了条件
- ラベルが「ターゲット基準:」に変わっていること
- 非選択状態のボタンにホバーすると背景色が変わること
- 「表示」ボタン群と同じホバー挙動であること
