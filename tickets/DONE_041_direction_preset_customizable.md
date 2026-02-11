# 041: 方面プリセット名を設定可能にする（ターゲット基準の右側に配置）

## 種別
UI改善（機能追加）

## 問題
方面選択のプリセットが「左」「正面」「右」で固定されているが、ステージや状況によって異なる呼び方をしたい。プリセットの3つの文言をユーザーが自由に設定できるようにしたい。

## 修正箇所
- `src/types/scenario.ts` — ScenarioState にプリセット名の設定を追加
- `src/hooks/scenarioReducer.ts` — アクション追加
- `src/ScenarioView.tsx` — 設定UIの追加（ターゲット基準の右側）
- `src/components/Timeline/DirectionLabels.tsx` — プリセット名を props で受け取る

## 修正内容

### 型の追加

```ts
// src/types/scenario.ts
export interface ScenarioState {
  // ... 既存フィールド
  readonly directionPresets: readonly [string, string, string]; // 方面選択肢の3文言
}
```

初期値: `['左', '正面', '右']`

### Reducer アクション追加

```ts
| { type: 'SET_DIRECTION_PRESET'; payload: { index: 0 | 1 | 2; name: string } }
```

### 設定UIの配置

ターゲット基準の右側に、3つの短いテキスト入力を横並びで配置:

```tsx
<div className="flex items-center gap-2">
  <span className="text-sm text-text-muted">方面名:</span>
  {state.directionPresets.map((preset, i) => (
    <input
      key={i}
      type="text"
      value={preset}
      onChange={(e) => handleSetDirectionPreset(i as 0 | 1 | 2, e.target.value)}
      className="w-16 rounded-sm border border-border bg-surface px-1 py-0.5 text-center text-xs text-text"
      maxLength={10}
    />
  ))}
</div>
```

### DirectionLabels への反映

`PRESET_NAMES` を定数ではなく props で受け取るように変更:

```tsx
interface DirectionLabelsProps {
  directions: readonly DirectionSetting[];
  presetNames: readonly [string, string, string];
  onUpdateName?: (index: number, name: string) => void;
}
```

フロート内のプリセットボタンのラベルが `presetNames` の値になる。

### エクスポート/インポートの対応

`directionPresets` を保存/復元対象に含める。インポート時にフィールドがなければデフォルト値 `['左', '正面', '右']` を使用。

## 完了条件
- 設定パネルに方面名の入力欄が3つ表示されること
- 入力した文言が方面選択フロートのボタンラベルに反映されること
- エクスポート/インポートで方面プリセット名が保存・復元されること
- デフォルト値が「左」「正面」「右」であること
