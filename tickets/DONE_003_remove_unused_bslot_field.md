# 003: InterpolatedHazardConfig の未使用フィールド bSlotSpawnerDecisionFrame を削除

## 種別
バグ（未使用コードによる誤解リスク）

## 問題
`InterpolatedHazardConfig` に `bSlotSpawnerDecisionFrame` フィールドが定義され、`buildHazardConfig` で計算されているが、実際には `calculateSpawns` 内で別途 `bSlotOpenFrame + SPAWN_WAIT_FRAMES` として独自計算しており、このフィールドは一度も参照されていない。

2つの異なる計算経路が存在すると、将来の変更時に片方だけ修正される危険がある。

計算上は同じ値になる:
- `bSlotBaseFrame - 184` = `(bSlotBaseFrame - 214) + 30` = `bSlotOpenFrame + 30`

## 修正箇所
- `src/types/computed.ts` — `InterpolatedHazardConfig` インターフェース
- `src/utils/calculations.ts` — `buildHazardConfig` 関数

## 修正内容
1. `InterpolatedHazardConfig` から `bSlotSpawnerDecisionFrame` を削除
2. `buildHazardConfig` から `bSlotSpawnerDecisionFrame` の計算を削除
3. テストファイル内の `hazardConfig` オブジェクトリテラルからも `bSlotSpawnerDecisionFrame` を削除

## 完了条件
- `bSlotSpawnerDecisionFrame` がコードベースのどこにも存在しないこと（grep確認）
- 既存テストが通ること
