# 073: 方面背景色をくっきり淡色（淡ピンク/淡イエロー/淡シアン）に変更する

## 種別
UI改善

## 修正箇所
- `src/index.css` — CSS カスタムプロパティ

## 修正内容

```css
/* 変更前 */
--color-dir-0: #fef9c3;  /* 薄い黄 */
--color-dir-1: #fee2e2;  /* 薄い赤 */
--color-dir-2: #dbeafe;  /* 薄い青 */

/* 変更後 */
--color-dir-0: #fdf2f8;  /* 淡ピンク (pink-50) */
--color-dir-1: #fefce8;  /* 淡イエロー (yellow-50) */
--color-dir-2: #ecfeff;  /* 淡シアン (cyan-50) */
```

## 完了条件
- 方面ID 0 が淡ピンク（#fdf2f8）で表示されること
- 方面ID 1 が淡イエロー（#fefce8）で表示されること
- 方面ID 2 が淡シアン（#ecfeff）で表示されること
