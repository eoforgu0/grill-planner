# 060: レーン幅を160pxに拡大し、マーカーを左1/3に配置する

## 種別
UI改善

## 問題
LANE_WIDTH = 120 ではターゲット付きラベルがはみ出す。マーカーがレーン中央にあるため、右側のラベル表示領域が半分しかない。

## 修正箇所
- `src/components/Timeline/coordinates.ts` — LANE_WIDTH 変更 + マーカー水平位置の定数追加
- `src/components/Timeline/SpawnMarker.tsx` — left 位置変更
- `src/components/Timeline/DefeatMarker.tsx` — left 位置変更
- `src/components/Timeline/ActivePeriod.tsx` — left 位置変更
- `src/components/Timeline/RespawnConnector.tsx` — left 位置変更

## 修正内容

### coordinates.ts

```ts
export const LANE_WIDTH = 160;
export const MARKER_CENTER_RATIO = 1 / 3; // マーカーの水平位置（レーン幅に対する比率）
```

### 各マーカー・バーの水平位置

現在 `left: '50%'` としている箇所を `left: `${MARKER_CENTER_RATIO * 100}%`` に変更:

```tsx
// SpawnMarker, DefeatMarker
style={{
  left: `${MARKER_CENTER_RATIO * 100}%`,  // 33.3%
  ...
}}
```

ActivePeriod, RespawnConnector も同様に `left: '50%'` → `left: `${MARKER_CENTER_RATIO * 100}%`` に変更。

### 操作説明カードの位置調整

LANE_WIDTH が 120→160 に増えた分、操作説明カードの `right` を調整。レーン領域の右端から十分に離れていること。

## 完了条件
- LANE_WIDTH が 160px であること
- マーカー（湧き・撃破）がレーン幅の1/3（約53px）の位置に配置されていること
- ラベルがレーン内に十分収まるか、少なくとも操作説明カードと干渉しないこと
- ActivePeriod, RespawnConnector もマーカーと同じ水平位置にあること
