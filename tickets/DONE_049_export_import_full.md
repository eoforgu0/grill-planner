# 049: エクスポート/インポートで全設定を完全に復元できるようにする

## 種別
バグ（機能不足）

## 問題
現在のエクスポート/インポートは ScenarioData をそのまま JSON にしているが、インポート時に一部のフィールドが欠落していても補完されず無視される可能性がある。全フィールドが確実に保存・復元されることを保証する。

## 対象フィールド（ScenarioData の全フィールド）

```ts
interface ScenarioData {
  hazardLevel: number;              // ✅ 対応済み
  directions: DirectionSetting[];   // ✅ 対応済み
  defeats: DefeatPoint[];           // ✅ 対応済み
  memo: {
    scenarioCode: string;           // ✅ 確認
    weapons: string[];              // ✅ 対応済み（RowId 変換含む）
    specials: string[];             // ✅ 対応済み（RowId 変換含む）
    targetOrder: {
      mode: TargetMode;             // ⚠️ 確認必要
      order: string[];              // ⚠️ 確認必要（ターゲット順の25行分）
    };
    snatchers: string;              // ⚠️ 確認必要
    freeNote: string;               // ⚠️ 確認必要
  };
  displayMode: DisplayMode;         // ✅ 対応済み
  directionPresets: [str, str, str]; // ⚠️ 確認必要
}
```

## 修正箇所
- `src/utils/fileIO.ts` — インポート時のフィールド補完ロジック

## 修正内容

### エクスポート
現在の実装で ScenarioData を丸ごと JSON にしているので問題なし。ターゲット順（後続チケット051で追加予定）が memo.targetOrder.order に保存されていれば自動的にエクスポートされる。

### インポート時のフィールド補完

各フィールドがファイルに存在しない場合のデフォルト値:

```ts
const defaults: ScenarioData = {
  hazardLevel: 100,
  directions: [],
  defeats: [],
  memo: {
    scenarioCode: '',
    weapons: [],
    specials: [],
    targetOrder: { mode: 'weapon', order: [] },
    snatchers: '',
    freeNote: '',
  },
  displayMode: 'both',
  directionPresets: ['左', '正面', '右'],
};

// インポート時に各フィールドをマージ
const mergedMemo = {
  ...defaults.memo,
  ...(isObject(scenario.memo) ? scenario.memo : {}),
};
const merged: ScenarioData = {
  ...defaults,
  ...scenario,
  memo: mergedMemo,
};
```

### targetOrder.order のバリデーション

order 配列の各要素は `'1P'|'2P'|'3P'|'4P'|'-'` であること。不正な値は `'-'` に変換:

```ts
const VALID_TARGETS = new Set(['1P', '2P', '3P', '4P', '-']);
if (Array.isArray(mergedMemo.targetOrder?.order)) {
  mergedMemo.targetOrder.order = mergedMemo.targetOrder.order.map((v: unknown) =>
    typeof v === 'string' && VALID_TARGETS.has(v) ? v : '-'
  );
}
```

## 完了条件
- 全フィールドがエクスポートファイルに含まれること
- フィールドが欠落したファイルをインポートしてもエラーにならず、デフォルト値で補完されること
- ターゲット順（order 配列）が正しく保存・復元されること
- directionPresets が正しく保存・復元されること
- freeNote, snatchers が正しく保存・復元されること
