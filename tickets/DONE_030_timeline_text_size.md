# 030: タイムライン内のテキストサイズを拡大

## 種別
UI改善

## 問題
タイムライン内の各種テキスト（マーカーの時刻ラベル、方面ラベル、時間軸の目盛りラベル等）のフォントサイズが全体的に小さすぎて視認性が悪い。

## 修正箇所
- `src/components/Timeline/SpawnMarker.tsx` — 時刻・方面ラベル
- `src/components/Timeline/DefeatMarker.tsx` — 時刻ラベル
- `src/components/Timeline/TimeAxis.tsx` — 目盛りラベル
- `src/components/Timeline/DirectionLabels.tsx` — 方面名テキスト
- `src/components/Timeline/GrillSlotLane.tsx` — 枠ラベル（「A枠」「B枠」）

## 修正内容
各テキストのフォントサイズを以下の目安で拡大:

| 要素 | 現在 | 変更後 |
|------|------|--------|
| マーカー時刻/方面ラベル | 9px | 11px |
| 時間軸の主目盛りラベル | 10px | 12px |
| 方面名テキスト | 12px(text-xs) | 13〜14px(text-sm相当) |
| 枠ラベル（A枠/B枠） | 12px(text-xs) | 13〜14px |
| 操作説明テキスト | 適宜 | 11〜12px |

具体的な値はレイアウトの崩れが起きない範囲で調整すること。

## 完了条件
- タイムライン内の全テキストが現在より視認しやすくなっていること
- テキストサイズの拡大によりレイアウトが崩れていないこと
