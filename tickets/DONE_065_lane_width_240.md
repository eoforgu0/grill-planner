# 065: タイムラインのレーン幅を240pxに拡大する

## 種別
UI改善

## 問題
ターゲット表示（テキスト + ブキアイコン）を加えると160pxでもはみ出しが大きい。

## 修正箇所
- `src/components/Timeline/coordinates.ts` — LANE_WIDTH

## 修正内容

```ts
export const LANE_WIDTH = 240;
```

操作説明カードの `right` 位置もレーン幅拡大に合わせて調整すること（Timeline/index.tsx）。

## 完了条件
- LANE_WIDTH が 240px であること
- 湧きマーカーのラベル + ブキアイコンがレーン内に概ね収まること
- 操作説明カードがレーンと干渉しないこと
