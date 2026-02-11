# 104: fileIO.ts のインポート時に撃破点の整合性チェックと部分除外を実装

## 種別
改善（仕様未実装）

## 問題
設計書 08_DATA_SPEC §3.4 に以下の記載がある:

> バリデーション通過後、さらに `calculateSpawns` + 撃破点の整合性チェックを行い、不整合な撃破点があれば警告メッセージと共にそれらを除外してインポートする（全拒否ではなく部分インポート）。

現在の `fileIO.ts` の `parseAndValidate` はJSONスキーマのチェックのみで、calculateSpawns による整合性チェックが未実装。キケン度を変更して保存→別のキケン度で読み込み、等のケースで不整合な撃破点が残ったままインポートされる。

## 修正箇所
- `src/utils/fileIO.ts`

## 修正内容
`parseAndValidate` のスキーマチェック通過後、以下の処理を追加:

1. `getHazardConfig` でキケン度から設定を取得
2. `calculateSpawns` で spawns を計算
3. 各 defeat について `isConsistentDefeat` で整合性チェック
4. 不整合な defeat を除外
5. 除外があった場合は `ImportResult` に警告メッセージを含める

`ImportResult` に `warnings?: string[]` フィールドを追加し、部分インポート時に除外された撃破点の情報を返す。呼び出し側（ScenarioView.tsx）で警告を表示する。

注意: `parseAndValidate` は現在 `HazardConfigData` を受け取っていないため、引数追加が必要。`importScenarioFromFile` の呼び出し側から渡す形にする。

## 完了条件
- インポート時に不整合な撃破点が自動除外されること
- 除外があった場合に警告メッセージが表示されること
- 整合性のあるデータはそのまま全量インポートされること
