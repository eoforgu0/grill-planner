# 009: 方面ラベル列の幅を広げる

## 種別
UI改善

## 問題
`DIRECTION_LABEL_WIDTH = 32px` ではデフォルト名「方面1」すら収まらず「方…」と省略表示される。

## 修正箇所
- `src/components/Timeline/coordinates.ts` — `DIRECTION_LABEL_WIDTH`
- `src/components/Timeline/DirectionLabels.tsx` — 必要に応じてフォントサイズ調整

## 修正内容
`DIRECTION_LABEL_WIDTH` を 56px 程度に拡大する。「左」「正面」「右」が十分に表示でき、かつ画面幅を圧迫しない値を選ぶこと。

⑬（方面ラベルの選択式UI）の修正と合わせて実施してよい。

## 完了条件
- デフォルト方面名（「左」「正面」「右」等）が省略されず完全に表示されること
