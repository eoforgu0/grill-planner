# 072: 操作説明カードのテキストが異常に折り返される問題を修正する

## 種別
バグ（表示）

## 問題
操作説明カードの幅が極端に狭くなり、テキストが1〜2文字ずつ折り返されている。

## 原因
操作説明カードは `position: absolute` + `left: lanesWidth + 20` で配置されている。その親要素（レーン領域の div）は `width: lanesWidth`（= 484px: 240*2+4）に設定されている。

absolute 配置された子要素は、親要素の content box 内で利用可能な幅をもとにテキストの折り返しを計算する。`left: lanesWidth + 20` だと、親の幅 `lanesWidth` から `left` を引いた残り幅が **負の値** になり、カード内のテキストが極限まで折り返される。

`whitespace: nowrap` や固定 `width` を指定していないため、ブラウザが自動的に最小幅で折り返しを行っている。

## 修正箇所
- `src/components/Timeline/index.tsx` — 操作説明カードのスタイル

## 修正内容

### 方針: `whiteSpace: 'nowrap'` を追加

操作説明カードのテキストは短い行の集まりで折り返しが不要なので、カード本体に `whiteSpace: 'nowrap'` を追加して強制的に折り返しを防ぐ:

```tsx
<div
  className="rounded-md border border-border px-3 py-2"
  style={{
    fontSize: 11,
    color: "var(--color-text-muted)",
    lineHeight: 1.6,
    backgroundColor: "var(--color-bg)",
    whiteSpace: "nowrap",        // ← 追加
  }}
>
```

これにより親要素の幅制約に関わらず、テキストが折り返されなくなる。

## 完了条件
- 操作説明カードのテキスト（「クリック: 撃破追加」等）が折り返されず1行で表示されること
- 凡例（「湧き」「撃破」）も正常に表示されること
- カードの位置（B枠右端+20px）は維持されること
