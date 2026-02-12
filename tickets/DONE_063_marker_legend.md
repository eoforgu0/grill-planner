# 063: 操作説明カードの下部にマーカー凡例を追加する

## 種別
UI改善

## 問題
初見ではひし形が撃破、丸が湧きであることがわからない。操作説明カード内にマーカーの凡例を追加する。

## 修正箇所
- `src/components/Timeline/index.tsx` — 操作説明カード内に凡例を追加

## 修正内容

操作説明の下に区切り線を入れ、マーカー凡例を追加:

```tsx
<div
  className="rounded-md border border-border px-3 py-2"
  style={{
    fontSize: 11,
    color: "var(--color-text-muted)",
    lineHeight: 1.6,
    backgroundColor: "var(--color-bg)",
  }}
>
  {/* 操作方法 */}
  <div className="mb-1 text-xs font-medium">操作方法</div>
  <div>クリック: 撃破追加</div>
  <div>ドラッグ: 撃破移動</div>
  <div>右クリック: 撃破削除</div>

  {/* 凡例 */}
  <div className="mt-2 border-t border-border pt-2">
    <div className="mb-1 text-xs font-medium">凡例</div>
    <div className="flex items-center gap-1.5">
      {/* 湧きマーカー: 緑丸 */}
      <div
        style={{
          width: 10,
          height: 10,
          borderRadius: '50%',
          backgroundColor: 'var(--color-spawn)',
          border: '1.5px solid var(--color-slot-a)',
          flexShrink: 0,
        }}
      />
      <span>湧き</span>
    </div>
    <div className="flex items-center gap-1.5">
      {/* 撃破マーカー: オレンジひし形 */}
      <div
        style={{
          width: 10,
          height: 10,
          backgroundColor: 'var(--color-defeat)',
          transform: 'rotate(45deg)',
          flexShrink: 0,
        }}
      />
      <span>撃破</span>
    </div>
  </div>
</div>
```

凡例のマーカーは実際のマーカーを縮小した形状で、色も実際と同じCSS変数を使用。

## 完了条件
- 操作説明カードの下部に「凡例」セクションがあること
- 緑丸 = 湧き、オレンジひし形 = 撃破 の凡例が表示されていること
- 凡例のマーカー形状・色が実際のタイムライン上のマーカーと一致すること
