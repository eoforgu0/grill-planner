# 091: 表示モードをLocalStorage管理に移行 + UI配置を右側に移動

## 種別
リファクタリング + UI改善

## 概要

表示モード（アイコン/テキスト/両方）を以下の2点で変更する:

1. **UI配置**: キケン度の横から、方面カラーの左に移動（ズーム等と同じ右寄せグループ）
2. **保存方式**: ScenarioData（エクスポート/インポート対象）から除外し、LocalStorage で保存・復元する

変更後のレイアウト:
```
キケン度: [100] %           表示: [アイコン][テキスト][両方]  方面カラー: [花▼]  ズーム: 横[100%▼] 縦[100%▼]
                            ←――――――――― ml-auto の右寄せグループ ――――――――――→
```

## 修正箇所

### 1. LocalStorage フック — `src/hooks/useZoom.ts` に追加

`useZoom.ts` に `useDisplayMode` を追加（ファイル名は `useZoom.ts` のまま維持。将来的に `useViewSettings.ts` にリネームしてもよい）:

```ts
import type { DisplayMode } from "@/types";

const STORAGE_KEY_DISPLAY_MODE = "grill-planner-display-mode";
const VALID_DISPLAY_MODES: readonly DisplayMode[] = ["icon", "text", "both"];
const DEFAULT_DISPLAY_MODE: DisplayMode = "both";

export function useDisplayMode() {
  const [displayMode, setDisplayModeState] = useState<DisplayMode>(() => {
    const saved = localStorage.getItem(STORAGE_KEY_DISPLAY_MODE);
    return saved && VALID_DISPLAY_MODES.includes(saved as DisplayMode)
      ? (saved as DisplayMode)
      : DEFAULT_DISPLAY_MODE;
  });

  const setDisplayMode = useCallback((mode: DisplayMode) => {
    setDisplayModeState(mode);
    localStorage.setItem(STORAGE_KEY_DISPLAY_MODE, mode);
  }, []);

  return { displayMode, setDisplayMode };
}
```

### 2. ScenarioData から displayMode を除去 — `src/types/scenario.ts`

```ts
export interface ScenarioData {
  readonly hazardLevel: number;
  readonly directions: readonly DirectionSetting[];
  readonly defeats: readonly DefeatPoint[];
  readonly memo: ScenarioMemo;
  // readonly displayMode: DisplayMode;  // 削除
  readonly directionPresets: readonly [string, string, string];
}
```

### 3. scenarioReducer — `src/hooks/scenarioReducer.ts`

- `SET_DISPLAY_MODE` アクションを削除
- `createInitialScenario` から `displayMode` フィールドを削除
- `LOAD_SCENARIO` で `displayMode` を扱わない

### 4. ScenarioView.tsx

- `useDisplayMode()` フック呼び出しを追加
- `handleDisplayModeChange` を useDisplayMode の `setDisplayMode` に置換
- `DisplayModeToggle` の配置を `ml-auto` の右寄せグループ内に移動
- Timeline への `displayMode` props は useDisplayMode から取得した値を渡す

```tsx
const { displayMode, setDisplayMode } = useDisplayMode();

// 設定パネル
<div className="flex flex-wrap items-center gap-6">
  <HazardLevelInput value={state.hazardLevel} onChange={handleHazardChange} />

  {/* 右寄せグループ */}
  <div className="ml-auto flex items-center gap-4 text-xs text-text-muted">
    <DisplayModeToggle value={displayMode} onChange={setDisplayMode} />
    {/* 方面カラー */}
    <label ...>方面カラー ...</label>
    {/* ズーム */}
    <span>ズーム:</span> ...
  </div>
</div>
```

### 5. エクスポート/インポート — `src/utils/fileIO.ts`

**エクスポート**: `ScenarioData` から `displayMode` がなくなるため、自動的にエクスポートされなくなる。変更不要。

**インポート**: `parseAndValidate` 内の `displayMode` の補完処理を削除:

```ts
// 削除:
// if (scenario.displayMode !== "icon" && ...) {
//   scenario.displayMode = "both";
// }
```

ただし、旧形式のファイル（`displayMode` フィールドを含む）をインポートしてもエラーにならないようにする。`ScenarioData` に含まれない余分なフィールドは単に無視される（`as unknown as ScenarioData` のキャスト時に無視）。

### 6. 定数 — `src/constants/index.ts`

`DEFAULT_DISPLAY_MODE` が定義されている場合、useDisplayMode 側で直接定義するか、定数を維持してインポートするか統一する。useZoom と同じパターンでフック内に閉じるのが望ましい。

## 後方互換性

- 旧形式のエクスポートファイル（`displayMode` フィールドを含む）は正常にインポート可能（余分なフィールドは無視）
- 既存の LocalStorage にはキーが存在しないため、初回は `DEFAULT_DISPLAY_MODE`（"both"）が使用される

## 完了条件
- 表示モードのトグルが方面カラーの左、右寄せグループ内に配置されていること
- 表示モードの切り替えが即座に反映されること
- ページリロード後に表示モードが復元されること
- エクスポートファイルに `displayMode` が含まれないこと
- インポート時に `displayMode` が無視されること（旧ファイルでもエラーにならないこと）
- LocalStorage に無効な値が入っていた場合、"both" にフォールバックすること
- `ScenarioData` 型から `displayMode` フィールドが除去されていること
- scenarioReducer から `SET_DISPLAY_MODE` アクションが除去されていること
