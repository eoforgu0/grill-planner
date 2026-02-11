# 011: リスポーン接続線が見えない問題を修正

## 種別
バグ（表示）

## 問題
撃破マーカー→スポナー決定◇→湧きマーカーの間に描画されるべきリスポーン接続線（RespawnConnector）が画面上で見えない。z-index の問題で活動期間バーや方面帯の下に隠れている可能性がある。

## 修正箇所
- `src/components/Timeline/RespawnConnector.tsx` — z-index の見直し
- `src/components/Timeline/GrillSlotLane.tsx` — 描画順・z-index の見直し

## 修正内容
各要素の z-index 整理:
1. 方面帯（背景）: z-index: 0
2. 活動期間バー: z-index: 1
3. リスポーン接続線: z-index: 2 〜 3（マーカーより下、バーより上）
4. 湧きマーカー: z-index: 3
5. 撃破マーカー: z-index: 4
6. ドラッグ中マーカー: z-index: 10

RespawnConnector の SVG コンテナに適切な z-index を設定し、overflow: visible を確保すること。

## 完了条件
- 撃破→スポナー決定→湧きの接続線（実線→◇→破線）が視認できること
- 他の要素（活動期間バー、方面帯）の上に描画されること
- Playwright で確認推奨
