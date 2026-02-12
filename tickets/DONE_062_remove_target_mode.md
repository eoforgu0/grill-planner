# 062: ターゲット基準の設定UIを削除する

## 種別
UI改善（機能削除）

## 問題
ターゲット基準（ブキ/プレイヤー）の設定は不要になった。ターゲット順テーブルで直接プレイヤーを選択するため、ブキ順/プレイヤー順という基準の切替は意味がなくなった。

## 修正箇所
- `src/ScenarioView.tsx` — ターゲット基準のUI削除、handleSetTargetMode 削除
- `src/types/scenario.ts` — targetOrder.mode フィールドの削除を検討
- `src/hooks/scenarioReducer.ts` — SET_TARGET_MODE アクションの削除を検討
- `src/types/base.ts` — TargetMode 型の削除を検討

## 修正内容

### ScenarioView からUI削除

設定パネルの以下を削除:
```tsx
// 削除
<div className="flex items-center gap-2">
  <span className="text-sm text-text-muted">ターゲット基準:</span>
  <ButtonGroup
    options={[
      { value: "weapon" as TargetMode, label: "ブキ" },
      { value: "player" as TargetMode, label: "プレイヤー" },
    ]}
    selected={state.memo.targetOrder.mode}
    onChange={handleSetTargetMode}
  />
</div>
```

`handleSetTargetMode` コールバックも削除。

### 型・Reducer のクリーンアップ

`targetOrder.mode` と `TargetMode` 型を使用している箇所が他にない場合:
- `ScenarioMemo.targetOrder.mode` を削除
- `TargetMode` 型を削除
- `SET_TARGET_MODE` アクションを削除

ただし `targetOrder` オブジェクト自体は `order` 配列を保持するため残す。構造を簡素化する場合:

```ts
// 変更前
readonly targetOrder: {
  readonly mode: TargetMode;
  readonly order: readonly string[];
};

// 変更後
readonly targetOrder: readonly string[];
```

このフラット化はエクスポート/インポートの互換性に影響するため、既存のファイルとの後方互換を考慮してインポート時に両形式を受け入れるようにすること。

## 完了条件
- 設定パネルからターゲット基準のUIが削除されていること
- 不要になった型・アクション・コールバックが削除されていること
- エクスポート/インポートが正常に動作すること
