# 102: defeatCounter をモジュールスコープから useRef に移動

## 種別
改善（StrictMode 安全性）

## 問題
`src/components/Timeline/index.tsx` でID生成用カウンタがモジュールスコープの `let` 変数として定義されている:

```ts
let defeatCounter = 0;
```

React の StrictMode では開発時にコンポーネントが二重にマウント/アンマウントされるため、モジュールスコープの可変変数は予期しない挙動を起こしうる。`Date.now()` との組み合わせで実用上は問題ないが、正しい React パターンに従うべき。

## 修正箇所
- `src/components/Timeline/index.tsx`

## 修正内容
```ts
// Before
let defeatCounter = 0;

// After
const defeatCounterRef = useRef(0);

// 使用箇所
const id = `defeat-${Date.now()}-${defeatCounterRef.current++}`;
```

## 完了条件
- モジュールスコープの `let defeatCounter` が削除されていること
- useRef によるカウンタに置き換えられていること
