# 051: 右ペインにターゲット順入力UIを追加する

## 種別
UI改善（機能追加）

## 目的
ターゲット順を入力することで、N体目のグリルが誰のターゲットなのかわかるようにする。

## ゲーム仕様
- ターゲット順は同一シナリオであれば固定の順番
- A枠・B枠合わせた合計でN体目の湧きのグリルのターゲットは、ターゲット順のN番目のプレイヤー

## UI 仕様

方面名別合計の下に配置:

```
ターゲット順                    ← タイトル
▽1つ下にずらす  △1つ上にずらす  ← シフトボタン

# | ターゲット
--|----------
1 | [1P🔫] [2P🔫] [3P🔫] [4P🔫] [-]   ← ButtonGroup
2 | [1P🔫] [2P🔫] [3P🔫] [4P🔫] [-]
3 | [1P🔫] [2P🔫] [3P🔫] [4P🔫] [-]
...
25| [1P🔫] [2P🔫] [3P🔫] [4P🔫] [-]
```

### 各行の ButtonGroup

- 選択肢: `1P`, `2P`, `3P`, `4P`, `-`
- 各P の右横にはブキ選択で選ばれているブキのアイコンを表示（16x16程度の小サイズ）
- ブキ未選択の場合はアイコン分の固定幅 div を表示して崩れを防止
- `-` はターゲット未定/未選択を示す（デフォルト状態）
- 050 の ButtonGroup と同じスタイルを使用

### 行数

25行固定。ゲーム内で理論上出現しうる最大グリル数をカバーするのに十分。

### ずらしボタン

```
▽1つ下にずらす: 全行の値を1つ下にシフト
  - 行1の値 → 破棄
  - 行2の値 → 行1
  - 行3の値 → 行2
  - ...
  - 行25の値 → 行24
  - 行25 → '-'（空から加わる）

△1つ上にずらす: 全行の値を1つ上にシフト
  - 行1の値 → 行2
  - 行2の値 → 行3
  - ...
  - 行24の値 → 行25
  - 行25の値 → 破棄
  - 行1 → '-'（空から加わる）
```

## 修正箇所

### 新規コンポーネント
- `src/components/Statistics/TargetOrderTable.tsx` — ターゲット順テーブル

### 型・状態管理
- `src/types/scenario.ts` — targetOrder.order の型を明確化（25要素の配列）
- `src/hooks/scenarioReducer.ts` — SET_TARGET_ORDER_ENTRY, SHIFT_TARGET_ORDER アクション追加

### 配置
- `src/ScenarioView.tsx` — 右ペインに TargetOrderTable を追加

## 修正内容

### targetOrder.order の仕様

```ts
// 25要素の配列。各要素は '1P'|'2P'|'3P'|'4P'|'-'
type TargetEntry = '1P' | '2P' | '3P' | '4P' | '-';

// memo.targetOrder.order: TargetEntry[]（長さ25）
```

初期値: `Array(25).fill('-')`

### Reducer アクション

```ts
| { type: 'SET_TARGET_ORDER_ENTRY'; payload: { index: number; value: TargetEntry } }
| { type: 'SHIFT_TARGET_ORDER'; payload: 'up' | 'down' }
```

SHIFT_TARGET_ORDER の実装:

```ts
case 'SHIFT_TARGET_ORDER': {
  const order = [...state.memo.targetOrder.order];
  if (action.payload === 'down') {
    // 先頭を破棄、末尾に '-' を追加
    order.shift();
    order.push('-');
  } else {
    // 末尾を破棄、先頭に '-' を追加
    order.pop();
    order.unshift('-');
  }
  return {
    ...state,
    memo: {
      ...state.memo,
      targetOrder: { ...state.memo.targetOrder, order },
    },
  };
}
```

### TargetOrderTable コンポーネント

```tsx
interface TargetOrderTableProps {
  order: readonly string[];          // 25要素
  weapons: readonly string[];        // memo.weapons（RowId の配列、4要素）
  weaponMaster: readonly WeaponMaster[];
  onSetEntry: (index: number, value: string) => void;
  onShift: (direction: 'up' | 'down') => void;
}
```

各行:
```tsx
<div className="flex items-center gap-1">
  <span className="w-6 text-right text-xs text-text-muted">{i + 1}</span>
  <ButtonGroup
    options={[
      { value: '1P', label: '1P', icon: weaponIcon1P },
      { value: '2P', label: '2P', icon: weaponIcon2P },
      { value: '3P', label: '3P', icon: weaponIcon3P },
      { value: '4P', label: '4P', icon: weaponIcon4P },
      { value: '-', label: '-' },
    ]}
    selected={order[i] ?? '-'}
    onChange={(v) => onSetEntry(i, v)}
  />
</div>
```

### ButtonGroup へのアイコン対応

050 で作成する ButtonGroup コンポーネントにアイコン表示対応を追加:

```tsx
interface ButtonGroupOption<T> {
  value: T;
  label: string;
  icon?: string | null;  // アイコンURL（null ならアイコンなし）
}
```

ボタン内の描画:
```tsx
<button ...>
  {option.label}
  {option.icon !== undefined && (
    <div className="ml-0.5 h-4 w-4 shrink-0">
      {option.icon && <img src={option.icon} alt="" className="h-4 w-4" />}
    </div>
  )}
</button>
```

icon が `null` のとき: ブキ未選択 → アイコン分のスペースだけ確保（空 div）
icon が `undefined` のとき: アイコン欄自体を表示しない（'-' 選択肢など）

### 右ペインのレイアウト

```tsx
{/* 右ペイン */}
<div className="w-70 shrink-0 overflow-y-auto border-l border-border bg-surface p-4">
  <DirectionStatsTable ... />
  <div className="mt-4 border-t border-border pt-4">
    <TargetOrderTable ... />
  </div>
</div>
```

右ペインの幅（w-70 = 280px）に25行のButtonGroupが収まるよう、各ボタンはコンパクトに:
- ボタンラベル: 「1P」等の短いテキスト + 16x16 アイコン
- ボタン padding: px-1 py-0.5
- フォントサイズ: text-xs（12px）

### エクスポート/インポート

order 配列は memo.targetOrder.order として ScenarioData に含まれるため、049 のエクスポート/インポートで自動的にカバーされる。

## 完了条件
- 方面名別合計の下にターゲット順テーブルが表示されること
- 25行の ButtonGroup が表示され、各行で 1P/2P/3P/4P/- を選択できること
- 選択済みのプレイヤーのブキアイコンがボタン内に表示されること（ブキ未選択時はスペースのみ）
- ▽/△ボタンでターゲット順を1つずつシフトできること
- 初期状態が全行 `-` であること
- エクスポート/インポートでターゲット順が保存・復元されること
