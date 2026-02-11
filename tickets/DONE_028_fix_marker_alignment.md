# 028: 撃破マーカー・湧きマーカー・接続線の位置がバラバラになっている問題を修正

## 種別
バグ（致命的）

## 問題
添付画像で確認できる通り、撃破マーカー（◆）・リスポーン接続線（縦線）・湧きマーカー（●）の水平位置がバラバラにずれている。本来は同じ枠内で垂直に一列に並ぶべき。

原因の推定:
- マーカーは `left: 50%` + `transform: translate(-50%, -50%)` でレーン中央に配置
- RespawnConnector の SVG は `x = LANE_WIDTH / 2` でレーン中央を計算
- ActivePeriod は `right: 4` で右端配置
- これらの基準が合っていない、またはレーン内のポジショニングコンテキストがずれている可能性

## 修正箇所
- `src/components/Timeline/SpawnMarker.tsx`
- `src/components/Timeline/DefeatMarker.tsx`
- `src/components/Timeline/RespawnConnector.tsx`
- `src/components/Timeline/ActivePeriod.tsx`
- `src/components/Timeline/GrillSlotLane.tsx` — 必要に応じて

## 修正内容
全要素のX座標基準を統一する:

1. マーカー（Spawn/Defeat）: レーン中央に配置。`left: 50%` + `transform: translateX(-50%)` で統一。Y座標は `transform: translateY(-50%)` で中央寄せ。
2. RespawnConnector の SVG: `x = LANE_WIDTH / 2` をそのまま使用（これはレーン中央）
3. ActivePeriod: マーカーの右側に配置。マーカーと重ならない位置。

**確認ポイント**: GrillSlotLane の `position: relative` が正しく設定されており、子要素の `position: absolute` の基準がすべて同じレーン div になっていることを確認すること。

## 完了条件
- 撃破マーカー、湧きマーカー、リスポーン接続線が垂直に一列に並ぶこと
- ActivePeriod バーがマーカーと重ならないこと
- Playwright で確認推奨
