# 026: 方面選択のデフォルト値を全区間「正面」にする

## 種別
UI改善

## 問題
`generateDefaultDirections` で生成されるデフォルト方面名が「方面1」「方面2」…になっている。実際のゲームでは全区間が特定の方面であるケースが多く、「正面」をデフォルトにする方が使いやすい。

## 修正箇所
- `src/utils/calculations.ts` — `generateDefaultDirections` 関数

## 修正内容

```ts
// 変更前
return times.map((frameTime, index) => ({
  frameTime,
  direction: `方面${index + 1}`,
}));

// 変更後
return times.map((frameTime) => ({
  frameTime,
  direction: '正面',
}));
```

## 完了条件
- 初期状態およびキケン度変更で方面数が変わった場合に、全区間のデフォルト名が「正面」になること
- 既に方面名を編集済みの場合は（方面数が変わらなければ）保持されること
