# 076: 方面カラーテーマ選択機能（ドロップダウン + LocalStorage保存）

## 種別
機能追加

## 概要

方面の背景色をプリセットから選べるドロップダウンを追加する。
ズーム設定と同様に、エクスポート/インポート対象外とし、LocalStorageで保存・復元する。

## UI配置

設定パネルの行、ズームの左に配置:

```
キケン度: [100] %   表示: [アイコン] [テキスト] [両方]      方面カラー: [桜・空・若葉▼]  ズーム: 横 [100%▼] 縦 [100%▼]
```

## カラーテーマ定義

| キー | 表示名 | 方面0 (--color-dir-0) | 方面1 (--color-dir-1) | 方面2 (--color-dir-2) |
|------|--------|----------------------|----------------------|----------------------|
| `flower` | 花 | `#fce4ec` (赤) | `#e3f2fd` (青) | `#e8f5e9` (緑) |
| `pastel` | パステル | `#fee2e2` (赤) | `#dbeafe` (青) | `#dcfce7` (緑) |
| `kasumi` | 霞 | `#fdf4f4` (赤) | `#f4f8fd` (青) | `#f4faf5` (緑) |

- 色の順序: 方面0=赤系、方面1=青系、方面2=緑系
- デフォルト: `flower`（花）
- 常にいずれかのテーマが選択された状態（「なし」の選択肢はない）

※ 元候補との対応: flower = E（赤青緑に反転）、pastel = C（赤青緑に反転）、kasumi = G（赤青緑に反転）

## 保存方式

### useColorTheme カスタムフック（`src/hooks/useZoom.ts` に追加）

既存の `useZoom.ts` に同居させる（ファイル名は `useZoom.ts` のまま、または `useViewSettings.ts` にリネームしても可）:

```ts
const STORAGE_KEY_COLOR_THEME = "grill-planner-color-theme";

type ColorThemeKey = "flower" | "pastel" | "kasumi";

const COLOR_THEMES: Record<ColorThemeKey, { label: string; colors: [string, string, string] }> = {
  flower: { label: "花",      colors: ["#fce4ec", "#e3f2fd", "#e8f5e9"] },
  pastel: { label: "パステル", colors: ["#fee2e2", "#dbeafe", "#dcfce7"] },
  kasumi: { label: "霞",      colors: ["#fdf4f4", "#f4f8fd", "#f4faf5"] },
};

const DEFAULT_COLOR_THEME: ColorThemeKey = "flower";

export function useColorTheme() {
  const [themeKey, setThemeKeyState] = useState<ColorThemeKey>(() => {
    const saved = localStorage.getItem(STORAGE_KEY_COLOR_THEME);
    return saved && saved in COLOR_THEMES ? (saved as ColorThemeKey) : DEFAULT_COLOR_THEME;
  });

  const setThemeKey = useCallback((key: ColorThemeKey) => {
    setThemeKeyState(key);
    localStorage.setItem(STORAGE_KEY_COLOR_THEME, key);
  }, []);

  const theme = COLOR_THEMES[themeKey];

  return { themeKey, setThemeKey, theme, COLOR_THEMES };
}
```

## CSS カスタムプロパティの適用方法

index.css の `--color-dir-0/1/2` はデフォルト値として残す（フォールバック用）。
実行時にカラーテーマの色を CSS カスタムプロパティとして上書きする。

### 方法: ScenarioView でインラインスタイルをルート要素に設定

```tsx
// ScenarioView.tsx
const { themeKey, setThemeKey, theme, COLOR_THEMES } = useColorTheme();

return (
  <div
    className="flex h-screen flex-col bg-bg"
    style={{
      "--color-dir-0": theme.colors[0],
      "--color-dir-1": theme.colors[1],
      "--color-dir-2": theme.colors[2],
    } as React.CSSProperties}
  >
    ...
  </div>
);
```

この方法なら DirectionBands, DirectionLabels, getDirectionColor 等の既存コードは **変更不要**。
CSS カスタムプロパティの値がカスケードで上書きされるため、`var(--color-dir-0)` を参照している全ての箇所に自動的に反映される。

## ドロップダウンUI

```tsx
{/* ScenarioView.tsx 設定パネル内、ズームの左 */}
<div className="ml-auto flex items-center gap-4 text-xs text-text-muted">
  <label className="flex items-center gap-1">
    方面カラー
    <select
      value={themeKey}
      onChange={(e) => setThemeKey(e.target.value as ColorThemeKey)}
      className="rounded-sm border border-border bg-surface px-1 py-0.5 text-xs text-text"
    >
      {Object.entries(COLOR_THEMES).map(([key, { label }]) => (
        <option key={key} value={key}>{label}</option>
      ))}
    </select>
  </label>

  {/* 既存のズームUI */}
  <span>ズーム:</span>
  ...
</div>
```

## 修正箇所

### 新規/修正ファイル
- `src/hooks/useZoom.ts` — `useColorTheme` フック追加（同居 or リネーム）
- `src/ScenarioView.tsx` — `useColorTheme` 呼び出し、ドロップダウンUI追加、ルート要素にCSS変数上書き

### 変更不要
- `src/index.css` — `--color-dir-0/1/2` はフォールバックとして残す（削除しない）
- `src/components/Timeline/coordinates.ts` — `getDirectionColor` はそのまま
- `src/components/Timeline/DirectionBands.tsx` — `var(--color-dir-N)` を参照しており変更不要
- `src/components/Timeline/DirectionLabels.tsx` — 同上
- エクスポート/インポート — 一切関与しない

## 完了条件

- ドロップダウンで「花」「パステル」「霞」が選択できること
- デフォルトが「花」であること
- 常にいずれかのテーマが選択されていること（「なし」の選択肢はない）
- 選択したテーマがタイムラインの方面帯・方面ラベルに即時反映されること
- ページリロード後にテーマが復元されること
- localStorage に無効な値が入っていた場合、「花」にフォールバックすること
- エクスポート/インポートにカラーテーマが含まれないこと
