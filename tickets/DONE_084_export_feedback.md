# 084: エクスポート完了フィードバック

## 種別
UI改善

## 概要

エクスポートボタン押下時に、ダウンロードが実行されたことをユーザーにフィードバックする。

## 修正箇所
- `src/components/Header.tsx`

## 修正内容

### 状態管理

```tsx
type ExportState = "idle" | "exporting" | "done";
const [exportState, setExportState] = useState<ExportState>("idle");
```

### ボタンの挙動

1. **idle**: 通常表示、クリック可能
2. **exporting**: ボタン非活性（`disabled`）、テキスト「エクスポート中...」
3. **done**: ボタン活性に戻る、ボタン下にフキダシ「エクスポート完了」を表示

### フロー

```tsx
const handleExport = useCallback(async () => {
  if (exportState !== "idle") return;
  setExportState("exporting");

  // exportScenario は同期的にダウンロードリンクをクリックするだけだが、
  // UI更新のため次フレームまで待つ
  await new Promise((resolve) => requestAnimationFrame(resolve));
  onExport?.();

  setExportState("done");
  setTimeout(() => setExportState("idle"), 2000);
}, [exportState, onExport]);
```

### ボタン UI

```tsx
<div className="relative">
  <button
    type="button"
    onClick={handleExport}
    disabled={exportState === "exporting"}
    className="rounded-sm border border-border bg-surface px-3 py-1 text-sm text-text hover:bg-bg disabled:cursor-not-allowed disabled:opacity-50"
  >
    {exportState === "exporting" ? "エクスポート中..." : "エクスポート"}
  </button>

  {/* 完了フキダシ */}
  {exportState === "done" && (
    <div
      className="absolute top-full right-0 mt-1 whitespace-nowrap rounded-sm bg-text px-2 py-1 text-xs text-surface"
      style={{ zIndex: 30 }}
    >
      エクスポート完了
    </div>
  )}
</div>
```

フキダシは2秒後に自動消失（`setTimeout` で `idle` に戻る）。

## 完了条件
- エクスポートボタン押下でボタンが一瞬非活性になること
- ダウンロード実行後にボタン下に「エクスポート完了」フキダシが表示されること
- フキダシが約2秒後に自動消失すること
- フキダシ表示中もボタンはクリック可能な状態に戻っていること
