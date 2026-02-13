# 075: タイムラインズーム機能（横・縦独立、LocalStorage保存）

## 種別
機能追加

## 概要

タイムラインの縦方向・横方向を独立してズームできる機能を追加する。
CSS `transform: scale()` ではなく、各要素のサイズ・座標を倍率に応じて再計算する方式。

## UI配置

設定パネルの `DisplayModeToggle`（表示: アイコン/テキスト/両方）と同じ行の右端に配置:

```
キケン度: [100] %   表示: [アイコン] [テキスト] [両方]          ズーム: 横 [100%▼] 縦 [100%▼]
```

- ドロップダウン（`<select>`）で選択肢: 50%, 75%, 100%, 150%, 200%
- デフォルト: 横100%, 縦100%

## 保存方式

### useZoom カスタムフック (`src/hooks/useZoom.ts`)

```ts
const STORAGE_KEY_X = "grill-planner-zoom-x";
const STORAGE_KEY_Y = "grill-planner-zoom-y";
const ZOOM_OPTIONS = [50, 75, 100, 150, 200] as const;
const DEFAULT_ZOOM = 100;

export function useZoom() {
  const [zoomX, setZoomXState] = useState(() => {
    const saved = localStorage.getItem(STORAGE_KEY_X);
    const val = saved ? Number(saved) : DEFAULT_ZOOM;
    return ZOOM_OPTIONS.includes(val as any) ? val : DEFAULT_ZOOM;
  });

  const [zoomY, setZoomYState] = useState(() => {
    const saved = localStorage.getItem(STORAGE_KEY_Y);
    const val = saved ? Number(saved) : DEFAULT_ZOOM;
    return ZOOM_OPTIONS.includes(val as any) ? val : DEFAULT_ZOOM;
  });

  const setZoomX = useCallback((val: number) => {
    setZoomXState(val);
    localStorage.setItem(STORAGE_KEY_X, String(val));
  }, []);

  const setZoomY = useCallback((val: number) => {
    setZoomYState(val);
    localStorage.setItem(STORAGE_KEY_Y, String(val));
  }, []);

  // 倍率（0.5〜2.0）
  const scaleX = zoomX / 100;
  const scaleY = zoomY / 100;

  return { zoomX, zoomY, scaleX, scaleY, setZoomX, setZoomY, ZOOM_OPTIONS };
}
```

- ScenarioData / エクスポート / インポートには **一切含めない**
- localStorage のキーは `grill-planner-zoom-x`, `grill-planner-zoom-y`

## ズーム対象の定数と適用方法

### 基本方針

`coordinates.ts` の定数はモジュールスコープの固定値として維持し、各コンポーネントが `scaleX` / `scaleY` をpropsで受け取って計算時に乗算する。

これにより coordinates.ts は変更不要で、各コンポーネントのみの修正で済む。

### 縦方向（scaleY）の影響

| 対象 | 現在の値 | ズーム後 |
|------|---------|---------|
| `TIMELINE_HEIGHT` | 1600px | `1600 * scaleY` |
| `frameToPixelY(frame)` | `seconds * PIXELS_PER_SECOND` | `seconds * PIXELS_PER_SECOND * scaleY` |
| `TIMELINE_PADDING` | 12px | `12 * scaleY`（最低8px） |

`frameToPixelY` は広範囲で使われているため、ズーム対応版のラッパー関数を用意する:

```ts
// 各コンポーネントで使用するヘルパー（coordinates.ts に追加）
export function scaledFrameToPixelY(frameTime: FrameTime, scaleY: number): number {
  return frameToPixelY(frameTime) * scaleY;
}

export function scaledPixelYToFrame(pixelY: number, scaleY: number): FrameTime {
  return pixelYToFrame(pixelY / scaleY);
}
```

### 横方向（scaleX）の影響

| 対象 | 現在の値 | ズーム後 |
|------|---------|---------|
| `LANE_WIDTH` | 240px | `240 * scaleX` |
| `LANE_SPACING` | 4px | `4 * scaleX`（最低2px） |
| `DIRECTION_LABEL_WIDTH` | 56px | `56 * scaleX`（最低40px） |
| `TIME_AXIS_WIDTH` | 28px | `28 * scaleX`（最低20px） |

### 両方に影響（min(scaleX, scaleY) or 個別）

| 対象 | 現在の値 | ズーム後 | 備考 |
|------|---------|---------|------|
| `MARKER_SIZE` | 14px | `14 * scaleX`（最低8px） | 横方向に連動 |
| `ACTIVITY_BAR_WIDTH` | 8px | `8 * scaleX`（最低4px） | 横方向に連動 |
| フォントサイズ（11px, 12px, text-xs） | 固定 | `base * min(scaleX, scaleY)`（最低9px） | 可読性確保のため下限設定 |
| ブキアイコン（28x28） | 28px | `28 * scaleX`（最低14px） | 横方向に連動 |
| スポナー決定マーク | `MARKER_SIZE * 0.25` | ズーム後MARKER_SIZEから自動算出 | |
| RespawnConnector strokeWidth | 2px | `2 * scaleX`（最低1px） | |

### ズーム対象外

- 操作説明・凡例カード: 固定サイズのまま（情報表示であり、タイムラインの一部ではない）
- 設定パネル（キケン度、表示モード、詳細）
- 統計サイドバー

## 修正箇所と影響範囲

### 新規ファイル
- `src/hooks/useZoom.ts`

### 修正ファイル

**coordinates.ts** — ヘルパー関数追加のみ:
- `scaledFrameToPixelY(frameTime, scaleY)` 追加
- `scaledPixelYToFrame(pixelY, scaleY)` 追加

**ScenarioView.tsx** — ズームUIとフック:
- `useZoom()` 呼び出し
- ズームドロップダウンUIを設定パネルに追加
- `scaleX`, `scaleY` を Timeline に props 渡し

**Timeline/index.tsx** — scaleX, scaleY を受け取って子に伝搬:
- props に `scaleX`, `scaleY` 追加
- `lanesWidth` の計算に scaleX 反映
- 操作説明カードの位置は `lanesWidth + 20` のまま（ズーム対象外）
- 各子コンポーネントに scaleX, scaleY を渡す

**TimeAxis.tsx**:
- `ticks` のモジュールスコープ事前計算を、コンポーネント内の `useMemo` に移動（scaleY依存）
- `TIMELINE_HEIGHT * scaleY`, `PIXELS_PER_SECOND * scaleY` で目盛り位置計算
- フォントサイズに `min(scaleX, scaleY)` 適用
- ホバー位置計算に scaleY 反映

**DirectionLabels.tsx**:
- `DIRECTION_LABEL_WIDTH * scaleX` で幅計算
- `TIMELINE_HEIGHT * scaleY`, `scaledFrameToPixelY` で高さ・位置計算
- フォントサイズに倍率適用

**DirectionBands.tsx**:
- `scaledFrameToPixelY` で位置計算（scaleY）

**GrillSlotLane.tsx**:
- `LANE_WIDTH * scaleX` でレーン幅
- `scaledFrameToPixelY` でクリック位置→フレーム変換
- `scaledPixelYToFrame` でドラッグ時のフレーム計算
- 灰色エリアの高さに scaleY 反映

**SpawnMarker.tsx**:
- `scaledFrameToPixelY` で Y 座標
- `MARKER_SIZE * scaleX` でマーカーサイズ
- フォントサイズに倍率適用
- ブキアイコンサイズに scaleX 適用

**DefeatMarker.tsx**:
- `scaledFrameToPixelY` で Y 座標
- `MARKER_SIZE * scaleX` でマーカーサイズ
- フォントサイズに倍率適用
- 編集用 input のサイズに scaleX 適用

**ActivePeriod.tsx**:
- `scaledFrameToPixelY` で Y 座標
- `ACTIVITY_BAR_WIDTH * scaleX` でバー幅

**RespawnConnector.tsx**:
- `scaledFrameToPixelY` で各 Y 座標
- `LANE_WIDTH * scaleX * MARKER_CENTER_RATIO` で X 座標
- strokeWidth, circle の r に scaleX 適用

## ズームUIコンポーネント

設定パネルの行に直接記述（小規模のため独立コンポーネント不要）:

```tsx
{/* ScenarioView.tsx 設定パネル内 */}
<div className="flex flex-wrap items-center gap-6">
  <HazardLevelInput value={state.hazardLevel} onChange={handleHazardChange} />
  <DisplayModeToggle value={state.displayMode} onChange={handleDisplayModeChange} />

  {/* 右端寄せ */}
  <div className="ml-auto flex items-center gap-2 text-xs text-text-muted">
    <span>ズーム:</span>
    <label className="flex items-center gap-1">
      横
      <select
        value={zoomX}
        onChange={(e) => setZoomX(Number(e.target.value))}
        className="rounded-sm border border-border bg-surface px-1 py-0.5 text-xs text-text"
      >
        {ZOOM_OPTIONS.map((v) => (
          <option key={v} value={v}>{v}%</option>
        ))}
      </select>
    </label>
    <label className="flex items-center gap-1">
      縦
      <select
        value={zoomY}
        onChange={(e) => setZoomY(Number(e.target.value))}
        className="rounded-sm border border-border bg-surface px-1 py-0.5 text-xs text-text"
      >
        {ZOOM_OPTIONS.map((v) => (
          <option key={v} value={v}>{v}%</option>
        ))}
      </select>
    </label>
  </div>
</div>
```

## 実装上の注意

### 下限制約

極端な縮小でUIが破綻しないよう、各要素にピクセル下限を設ける:

```ts
function clampMin(value: number, min: number): number {
  return Math.max(value, min);
}

// 使用例
const markerSize = clampMin(MARKER_SIZE * scaleX, 8);
const fontSize = clampMin(11 * Math.min(scaleX, scaleY), 9);
```

### GrillSlotLane のドラッグ操作

ドラッグ中のフレーム計算に scaleY を反映する必要がある:
- `pixelYToFrame` を `scaledPixelYToFrame(pixelY, scaleY)` に置換
- クリック追加時の Y→フレーム変換も同様

### TimeAxis のモジュールスコープ計算

現在 `ticks` はモジュールスコープで事前計算されている。scaleY に依存するため `useMemo` に移動が必要:

```ts
const scaledTicks = useMemo(() => {
  const result: Tick[] = [];
  for (let s = 0; s <= GAME_DURATION_SECONDS; s++) {
    const pixelY = (GAME_DURATION_SECONDS - s) * PIXELS_PER_SECOND * scaleY;
    let type: Tick["type"] = "micro";
    if (s % 10 === 0) type = "major";
    else if (s % 5 === 0) type = "minor";
    result.push({ second: s, pixelY, type });
  }
  return result;
}, [scaleY]);
```

### 操作説明カードの位置

ズーム後の `lanesWidth`（`(LANE_WIDTH * scaleX * 2 + LANE_SPACING * scaleX)`）に追従する。
カード自体のサイズ・フォントはズーム対象外のまま。

## 完了条件

- `useZoom` フックが localStorage から保存・復元すること
- ドロップダウンで 50%/75%/100%/150%/200% が選択できること
- 横ズームでレーン幅、マーカーサイズ、アイコンサイズ等が正しくスケールすること
- 縦ズームでタイムラインの高さ、目盛り間隔、マーカー位置等が正しくスケールすること
- 50%でも100%でも200%でも表示が破綻しないこと（下限制約）
- ドラッグ操作がズーム後も正確にフレーム変換されること
- クリック追加がズーム後も正確にフレーム変換されること
- 操作説明カードはズーム対象外で、位置のみレーン幅に追従すること
- エクスポート/インポートにズーム値が含まれないこと
- ページリロード後にズーム値が復元されること
- localStorage に無効な値が入っていた場合、100%にフォールバックすること
