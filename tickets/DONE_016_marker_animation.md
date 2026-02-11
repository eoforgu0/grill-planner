# 016: マーカーの追加/削除時のアニメーションを実装

## 種別
UI改善

## 問題
設計書 06_UI_DESIGN §9 で定義されたマーカーの追加時フェードイン（0→1, 150ms ease-out）・削除時フェードアウト（1→0, 150ms ease-in）が未実装。マーカーの出現・消失が瞬間的で操作の因果関係がわかりにくい。

## 修正箇所
- `src/components/Timeline/SpawnMarker.tsx`
- `src/components/Timeline/DefeatMarker.tsx`

## 修正内容
マーカーのマウント時に CSS アニメーションでフェードイン + スケールアップを適用する。

追加時:
```css
@keyframes marker-in {
  from { opacity: 0; transform: translate(-50%, -50%) scale(0.5); }
  to   { opacity: 1; transform: translate(-50%, -50%) scale(1); }
}
```

削除時のアニメーションは React でのアンマウント時アニメーションが複雑なため、優先度低。追加時のフェードインのみの実装でも可。

設計書の方針に従い、`transform` と `opacity` のみをアニメーション対象とし、`width`, `height`, `top`, `left` のアニメーションは避けること。

## 完了条件
- マーカー追加時にフェードイン + スケールアップのアニメーション（150ms程度）が再生されること
- アニメーションが操作性を阻害しないこと（150ms以内）
