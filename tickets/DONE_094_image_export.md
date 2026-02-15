# 094: 画像出力機能

## 種別
機能追加

## 概要

現在のシナリオを1枚のPNG画像として出力する機能を追加する。
タイムラインが縦長でスクリーンキャプチャが困難なため、画像出力専用のレイアウトで描画し、
ツールを開かなくても内容が伝わる画像を生成する。

## 実装方針

画像出力専用のReactコンポーネント（非表示）をレンダリングし、html2canvas でキャプチャしてPNG出力する。

### ライブラリ
- `html2canvas` (`npm install html2canvas`)

### 出力フロー
1. ヘッダーの「画像出力」ボタンをクリック
2. 画像出力専用コンポーネントを非表示の固定サイズコンテナにレンダリング
3. html2canvas でキャプチャ
4. Canvas → Blob → ダウンロードリンク → PNG保存
5. 非表示コンテナを除去

## 画像レイアウト

```
┌──────────────────────────────────────────────────────────────────┐
│ Grill Planner                              キケン度: 100%        │
├───────────────────────────────────┬──────────────────────────────┤
│                                  │  ■ 詳細                       │
│  方   時                         │  コード: SXXX-XXXX-XXXX-XXXX  │
│  面   間   [A枠]      [B枠]     │  ブキ: 1P .96ガロン  2P ...   │
│  ラ   軸                灰色     │  SP:   1P ナイスダマ  2P ...   │
│  ベ                    エリア    │  タマヒロイ方向: 正面           │
│  ル                     ┌────┐  │  メモ: xxxxx                   │
│       100s ○  ◇       │凡例 │  │                                │
│                        │    │  │  ■ 方面別統計                   │
│        90s ○  ◇       │    │  │  # 方面  湧き 撃破              │
│                        └────┘  │  1 正面   2    2                │
│        80s ○    ○             │  2 左     2    1                 │
│                                │  ...                            │
│        70s ○  ◇  ◇           │  合計    8    6                  │
│                                │                                 │
│        60s                     │  ■ 方面名別合計（該当時のみ）    │
│            ...                 │                                 │
│         0s                     │                                 │
├───────────────────────────────────┴──────────────────────────────┤
│ YYYY/MM/DD HH:MM 生成                                            │
└──────────────────────────────────────────────────────────────────┘
```

### 凡例の配置

B枠の無効エリア（灰色）内に凡例を配置する。

- B枠が存在する場合: B枠レーンの灰色エリア内（上部、100s〜B枠開放秒の間）
- B枠が存在しない場合: タイムラインの右上（操作説明カードの位置に相当する場所）

凡例の内容（操作方法は除外、凡例のみ）:

```
凡例
  ● 湧き
  ◆ 撃破
  ◉ スポナー確定
  ⬡ 湧き（抑制）
```

Web表示の凡例と同じマーカー形状・色を使用するが、サイズは画像出力に合わせて調整。

### ヘッダー行

```
Grill Planner                              キケン度: XXX%
```

左にタイトル、右にキケン度。ズーム・方面カラー等の設定UIは含めない。

### タイムライン部分

Web表示と同じ要素を描画（ズーム100%固定）:

- 方面ラベル列
- 時間軸（100s, 90s, ...）
- 方面帯背景色（カラーテーマ反映）
- A枠レーン（湧きマーカー、撃破マーカー、リスポーン接続線、活動期間バー、経過時間ラベル）
- B枠レーン（同上 + 無効エリア灰色 + 凡例）
- 操作説明カードは**非表示**

### 右パネル — 詳細

入力がないフィールドは行ごと省略:

| フィールド | 省略条件 |
|-----------|---------|
| シナリオコード | 空文字列 |
| ブキ | 4人とも未設定 |
| SP | 4人とも未設定 |
| タマヒロイ方向 | 空文字列 |
| メモ | 空文字列 |

ブキ/SPは未設定の個人は「-」表示。アイコンも表示する（Web表示と同じ）。

### 右パネル — 方面別統計

Web表示の DirectionStatsTable と同じ内容:

- 方面切替ごとの湧き数・撃破数テーブル
- 合計行
- 方面名別合計（同名方面が複数区間ある場合のみ表示）

### フッター行

```
YYYY/MM/DD HH:MM 生成
```

生成日時を小さく表示。

## 新規ファイル

### `src/components/ImageExport/ExportRenderer.tsx`

画像出力専用のレイアウトコンポーネント。

Props:
```ts
interface ExportRendererProps {
  hazardLevel: number;
  spawns: readonly SpawnPoint[];
  defeats: readonly DefeatPoint[];
  directions: readonly DirectionSetting[];
  hazardConfig: InterpolatedHazardConfig;
  directionPresets: readonly [string, string, string];
  directionStats: readonly DirectionStats[];
  totalGrillCount: number;
  memo: ScenarioMemo;
  weapons: readonly WeaponMaster[];
  specials: readonly SpecialMaster[];
  targetOrder: readonly string[];
  weaponRowIds: readonly string[];
  displayMode: DisplayMode;
  colorThemeVars: Record<string, string>;  // CSS変数のインライン上書き用
}
```

描画はWeb表示のコンポーネントを流用せず、静的な表示専用として構築する。
ただし座標計算（coordinates.ts）やマーカー形状は共通のものを使用。

固定幅（例: 1200px）で描画し、html2canvas のスケールで解像度を調整する。

### `src/utils/imageExport.ts`

画像出力のユーティリティ:

```ts
export async function exportAsImage(container: HTMLElement, filename?: string): Promise<void> {
  const { default: html2canvas } = await import("html2canvas");
  const canvas = await html2canvas(container, {
    scale: 2,  // Retina 相当の解像度
    backgroundColor: "#ffffff",
    useCORS: true,  // ブキアイコン読み込み用
  });
  const blob = await new Promise<Blob | null>((resolve) => canvas.toBlob(resolve, "image/png"));
  if (!blob) return;
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = filename ?? `grill-plan-${formatDate()}.png`;
  a.click();
  URL.revokeObjectURL(url);
}
```

## 既存ファイルの修正

### `src/components/Header.tsx`

「画像出力」ボタンを追加（エクスポート/インポートの横）:

```tsx
<button
  type="button"
  onClick={onImageExport}
  className="... active:scale-95 transition-transform duration-100"
>
  画像出力
</button>
```

### `src/ScenarioView.tsx`

- ExportRenderer の非表示レンダリング
- 画像出力ハンドラ:

```tsx
const exportContainerRef = useRef<HTMLDivElement>(null);
const [isExporting, setIsExporting] = useState(false);

const handleImageExport = useCallback(async () => {
  setIsExporting(true);
  // 次フレームでDOMが更新されるのを待つ
  await new Promise(resolve => requestAnimationFrame(resolve));
  if (exportContainerRef.current) {
    await exportAsImage(exportContainerRef.current);
  }
  setIsExporting(false);
}, []);

// レンダリング（非表示）
{isExporting && (
  <div
    ref={exportContainerRef}
    style={{ position: "absolute", left: "-9999px", top: 0 }}
  >
    <ExportRenderer
      hazardLevel={state.hazardLevel}
      spawns={spawns}
      defeats={state.defeats}
      directions={state.directions}
      hazardConfig={hazardConfig}
      directionPresets={state.directionPresets}
      directionStats={directionStats}
      totalGrillCount={totalGrillCount}
      memo={state.memo}
      weapons={weapons}
      specials={specials}
      targetOrder={state.memo.targetOrder}
      weaponRowIds={state.memo.weapons}
      displayMode={displayMode}
      colorThemeVars={theme}
    />
  </div>
)}
```

### `package.json`

```
npm install html2canvas
```

## サイズ・解像度

- 描画コンテナ幅: 1200px（固定）
- 描画コンテナ高さ: タイムライン高さ（1600px + パディング）+ ヘッダー + フッター ≒ 1700px 程度
- html2canvas の scale: 2（出力画像は 2400×3400px 程度、Retina品質）
- 出力形式: PNG

## 完了条件
- ヘッダーに「画像出力」ボタンが表示されること
- ボタン押下でPNG画像がダウンロードされること
- 画像にタイムライン（方面ラベル・時間軸・マーカー・接続線・方面帯背景・経過時間ラベル）が含まれること
- 画像に凡例がB枠の灰色エリア内に表示されること（B枠なしの場合はタイムライン右上）
- 画像にキケン度が表示されること
- 画像に詳細（入力済みフィールドのみ）が表示されること
- 画像に方面別統計が表示されること
- 画像に操作説明カードが含まれないこと
- 画像にインタラクティブUI（ボタン・ドロップダウン等）が含まれないこと
- 画像の方面帯背景色がカラーテーマを反映すること
- フッターに生成日時が表示されること
- ブキ/SPアイコンが画像内に正しく表示されること（CORS対応）
