# 082: 方面色枠線の撤去（081の差し戻し）

## 種別
差し戻し

## 理由

チケット081で追加した方面色枠線は、カラーテーマの色が淡すぎてフキダシの枠線としてほぼ認識できなかった。
実質的に無意味な機能はコードの複雑化を招くだけなので撤去する。

## 差し戻し対象

081で変更した全ファイルを変更前の状態に戻す:

### SpawnMarker.tsx
- フキダシの `border: 1px solid ${directionColor}` を削除
- ブキアイコンの `border` を `1px solid var(--color-border)` に戻す
- `getDirectionColor` のインポートを削除（081で追加した場合）

### DefeatMarker.tsx
- `directionColor` props を削除
- フキダシの `border` を削除（081以前は枠線なし）
- 編集モード input の `border` を `1px solid var(--color-border)` 等に戻す（元の状態）

### GrillSlotLane.tsx
- `defeatDirectionMap` の useMemo を削除
- DefeatMarker への `directionColor` props 渡しを削除
- `getDirectionColor` のインポートを削除（081で追加した場合）

### ElapsedTimeLabel.tsx（080で新規作成）
- `directionColor` props を削除
- フキダシの `border` を削除するか、`1px solid var(--color-border)` に変更
- 編集モード input も同様

## 完了条件
- 全フキダシに方面色枠線が表示されないこと
- 081で追加した `directionColor` 関連のprops・ロジック・インポートが全て除去されていること
- `npm run lint` で未使用インポートの警告が出ないこと
