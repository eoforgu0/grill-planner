# 064: 湧きマーカーのブキアイコンをフキダシ外に分離表示する

## 種別
UI改善

## 問題
ブキアイコンがフキダシ（テキストラベル）内にあるため、アイコン表示時にフキダシの高さが太くなる。アイコンをフキダシの右外に分離し、独立した背景付きで表示する。

## 修正箇所
- `src/components/Timeline/SpawnMarker.tsx`

## 修正内容

フキダシ内からアイコンを除去し、フキダシの右に独立した要素として配置:

```tsx
return (
  <div className="absolute flex items-center" style={{ ... }}>
    {/* マーカー円 */}
    <div className="shrink-0 rounded-full" style={{ ... }} />

    {/* フキダシ（時刻+方面+ターゲットテキスト）— アイコンは含まない */}
    <span
      className="inline-flex items-center select-none whitespace-nowrap"
      style={{
        marginLeft: 4,
        fontSize: 11,
        color: "var(--color-text-muted)",
        backgroundColor: "rgba(255,255,255,0.85)",
        padding: "1px 4px",
        borderRadius: 2,
        lineHeight: 1.3,
      }}
    >
      {seconds}s {dirName}
      {targetLabel && displayMode !== "icon" && <> {targetLabel}</>}
    </span>

    {/* ブキアイコン（フキダシの右外、独立背景付き） */}
    {targetIcon && displayMode !== "text" && (
      <div
        style={{
          marginLeft: 3,
          padding: 2,
          backgroundColor: "rgba(255,255,255,0.85)",
          border: "1px solid var(--color-border)",
          borderRadius: 4,
          lineHeight: 0,
        }}
      >
        <img
          src={targetIcon}
          alt=""
          style={{ width: 28, height: 28, display: "block" }}
        />
      </div>
    )}
  </div>
);
```

ポイント:
- フキダシ（span）にはテキストのみ → 高さが一定
- アイコンは別の div で表示、`backgroundColor: "rgba(255,255,255,0.85)"` + `border: "1px solid var(--color-border)"` で薄灰色縁取りの角丸四角背景
- `borderRadius: 4` で角丸
- `lineHeight: 0` + `display: "block"` で img 周囲の余分なスペースを除去

## 完了条件
- ブキアイコンがフキダシの右外に表示されること
- ブキアイコンの背景が半透明白色で薄灰色の縁取りの角丸四角であること
- アイコン非表示時（未選択 or displayMode=text）にフキダシの高さが変わらないこと
