# 103: 定数に as const を統一

## 種別
改善（型の一貫性）

## 問題
`src/constants/index.ts` で、ほとんどの定数に `as const` が付いているが、`RESPAWN_FRAMES` と `GAME_DURATION_FRAMES` だけ付いていない:

```ts
export const RESPAWN_FRAMES = 214;       // ← as const なし
export const GAME_DURATION_FRAMES = 6000; // ← as const なし
```

型の狭小化（リテラル型推論）の一貫性が崩れている。

## 修正箇所
- `src/constants/index.ts`

## 修正内容
```ts
export const RESPAWN_FRAMES = 214 as const;
export const GAME_DURATION_FRAMES = 6000 as const;
```

## 完了条件
- 全定数に `as const` が付いていること
- ビルドが通ること
