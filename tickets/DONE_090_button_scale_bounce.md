# 090: エクスポート/インポートボタンにスケールバウンスエフェクト追加

## 種別
UI改善

## 概要

ヘッダーのエクスポート・インポートボタンにスケールバウンス（押し込み感）のフィードバックエフェクトを追加する。

## 修正箇所
- `src/components/Header.tsx`

## 修正内容

両ボタンの className に `active:scale-95 transition-transform duration-100` を追加:

```tsx
<button
  type="button"
  onClick={handleExport}
  disabled={exportState === "exporting"}
  className="rounded-sm border border-border bg-surface px-3 py-1 text-sm text-text
    hover:bg-bg active:scale-95 transition-transform duration-100
    disabled:cursor-not-allowed disabled:opacity-50"
>
  ...
</button>

<button
  type="button"
  onClick={onImport}
  className="rounded-sm border border-border bg-surface px-3 py-1 text-sm text-text
    hover:bg-bg active:scale-95 transition-transform duration-100"
>
  インポート
</button>
```

`:active` 擬似クラスでボタンを95%に縮小し、離した瞬間に100%に戻る。
`transition-transform duration-100` で100msのなめらかなアニメーション。

## 完了条件
- エクスポートボタンをクリック（マウスダウン）した瞬間にボタンがわずかに縮小すること
- インポートボタンをクリック（マウスダウン）した瞬間にボタンがわずかに縮小すること
- マウスアップでボタンが元のサイズに戻ること
- 遷移がなめらか（100ms）であること
