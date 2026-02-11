# 042: ヘッダーをスクロール追従に固定し、右ペインをメインコンテンツと独立スクロールにする

## 種別
UI改善（レイアウト変更）

## 問題
1. ヘッダー（タイトル + エクスポート/インポート）がスクロールで画面外に消える。常に画面上部に固定表示したい。
2. 右ペイン（方面別統計）が左ペインのスクロールに連動してスクロールする（または画面全体がスクロールして右ペインが一緒に動く）。右ペインは独立したサイドバーとして、左ペインのスクロールに影響されないようにしたい。

## 修正箇所
- `src/ScenarioView.tsx` — 全体レイアウト構造の変更
- `src/components/Header.tsx` — sticky 対応（必要に応じて）

## 修正内容

### レイアウト構造

```
[header — sticky top, z-index: 50                    ]
[left pane (overflow-y: auto)  | right pane (overflow-y: auto)]
[  設定パネル                   | 方面別統計                    ]
[  タイムライン                 |                              ]
[                              |                              ]
```

具体的な CSS 構造:

```tsx
<div className="flex h-screen flex-col bg-bg">
  {/* ヘッダー — 固定 */}
  <Header ... className="sticky top-0 z-50 shrink-0" />

  {/* メインエリア — ヘッダー下の残り高さ全体 */}
  <div className="flex min-h-0 flex-1">
    {/* 左ペイン — 独立スクロール */}
    <div className="flex-1 overflow-y-auto">
      {/* 設定パネル */}
      <div className="border-b border-border bg-surface px-4 py-3">
        ...
      </div>
      {/* タイムライン */}
      <div className="p-4">
        <Timeline ... />
      </div>
    </div>

    {/* 右ペイン — 独立スクロール */}
    <div className="w-70 shrink-0 overflow-y-auto border-l border-border bg-surface p-4">
      <DirectionStatsTable ... />
    </div>
  </div>
</div>
```

### ポイント

1. **h-screen + flex-col**: ルートを画面高さに固定
2. **ヘッダーは shrink-0**: 高さ固定（48px）、スクロールに追従（常に画面上部）
3. **メインエリアは min-h-0 + flex-1**: 残り高さを占有
4. **左ペイン overflow-y: auto**: 設定パネル + タイムラインが長い場合に独立スクロール
5. **右ペイン overflow-y: auto**: 統計テーブルが長い場合に独立スクロール
6. **左右が独立**: 左をスクロールしても右は動かない、逆も然り

### I/O エラー/警告表示の配置
ヘッダーと左ペインの間に表示する場合、ヘッダーの shrink-0 の直下に置く。またはヘッダー内に組み込む。

## 完了条件
- ヘッダーが常に画面上部に表示されること（スクロールしても消えない）
- 左ペイン（設定 + タイムライン）をスクロールしても右ペインが動かないこと
- 右ペインをスクロールしても左ペインが動かないこと
- タイムラインの高さが画面に収まらない場合、左ペインで独立してスクロールできること
