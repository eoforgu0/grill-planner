# 095: 画像出力の改善（フィードバック・キケン度位置・ブキSPレイアウト・表示崩れ修正）

## 種別
UI改善 + バグ修正

## 概要

チケット094で実装した画像出力機能について、以下4点を修正する。

## 1. 画像出力完了フィードバック

エクスポートボタンと同様に、画像出力ボタンにも完了フィードバックを追加する。

### 修正箇所: `src/components/Header.tsx`

画像出力ボタンにもエクスポートと同じステートマシンを適用:

```ts
type ImageExportState = "idle" | "exporting" | "done";
const [imageExportState, setImageExportState] = useState<ImageExportState>("idle");
```

- idle: 通常表示
- exporting: ボタン非活性、テキスト「画像出力中...」
- done: ボタン下にフキダシ「画像出力完了」を2秒表示

Header の `onImageExport` を async 対応にし、完了を待ってから done に遷移:

```tsx
const handleImageExport = useCallback(async () => {
  if (imageExportState !== "idle") return;
  setImageExportState("exporting");
  await onImageExport?.();
  setImageExportState("done");
  setTimeout(() => setImageExportState("idle"), 2000);
}, [imageExportState, onImageExport]);
```

Header の props: `onImageExport?: () => Promise<void>`（戻り値を Promise に変更）

## 2. キケン度の表示位置変更

### 現状
ヘッダー行の右端に「キケン度: 120%」と表示。

### 変更後
ヘッダー行からキケン度を削除し、右ペインの最上部（「詳細」見出しの上）に移動。

```
右ペイン:
┌─────────────────────────────┐
│ キケン度: 120%               │  ← 大きめフォント、太字
│                              │
│ ■ 詳細                       │
│ コード: SXXX-XXXX-...        │
│ ...                          │
```

ヘッダー行は「Grill Planner」タイトルのみ（左寄せ）。

## 3. ブキ/SP レイアウトを表形式に変更

### 現状
ブキとSPが横並びテキストで表示され、長いブキ名で折り返しが崩れている。

### 変更後
4行×3列の表形式:

```
┌────┬──────┬──────────────────┐
│ 1P │ ブキ │ [icon] わかばシューター │
│    │ SP   │ [icon] ナイスダマ       │
├────┼──────┼──────────────────┤
│ 2P │ ブキ │ [icon] LACT-450        │
│    │ SP   │ [icon] トリプルトルネード │
├────┼──────┼──────────────────┤
│ 3P │ ブキ │ [icon] ジムワイパー      │
│    │ SP   │ [icon] ウルトラチャクチ   │
├────┼──────┼──────────────────┤
│ 4P │ ブキ │ [icon] フルイドV        │
│    │ SP   │ [icon] ホップソナー      │
└────┴──────┴──────────────────┘
```

- 1列目: プレイヤーID（1P〜4P）、ブキ行にのみ表示、SP行は空
- 2列目: 「ブキ」/「SP」ラベル
- 3列目: アイコン + 名前（未設定の場合は「-」）
- 罫線はCSSのborderで表現（薄いborder-border色）
- 各行は同じプレイヤーの中でブキ→SPの順

### 省略ルール
- ブキ: 4人とも未設定（全て空文字列）→ ブキ表全体を非表示
- SP: 4人とも未設定（全て空文字列）→ SP表全体を非表示
- 1人でも設定されていれば、未設定のプレイヤーは「-」表示

ただし、ブキとSPは1つのテーブルに統合して表示する（別々の表ではない）。
「どれか1つでもブキかSPが選択されていたら」テーブル全体を表示:

```ts
const hasAnyWeaponOrSpecial =
  weaponRowIds.some(id => id !== "") || specialRowIds.some(id => id !== "");
```

### 修正箇所: `src/components/ImageExport/ExportRenderer.tsx`

ブキ/SP表示部分を `<table>` に書き換え:

```tsx
{hasAnyWeaponOrSpecial && (
  <table style={{ fontSize: 12, borderCollapse: "collapse", width: "100%" }}>
    {PLAYER_IDS.map((pid, i) => {
      const weaponName = resolveWeaponName(weaponRowIds[i]);
      const weaponIcon = resolveWeaponIcon(weaponRowIds[i]);
      const specialName = resolveSpecialName(specialRowIds[i]);
      const specialIcon = resolveSpecialIcon(specialRowIds[i]);
      return (
        <Fragment key={pid}>
          <tr>
            <td rowSpan={2} style={{ padding: "2px 6px", borderBottom: "1px solid var(--color-border)", verticalAlign: "top", fontWeight: "bold" }}>
              {pid}
            </td>
            <td style={{ padding: "2px 4px", color: "var(--color-text-muted)" }}>ブキ</td>
            <td style={{ padding: "2px 4px", display: "flex", alignItems: "center", gap: 4 }}>
              {weaponIcon && <img src={weaponIcon} alt="" style={{ width: 20, height: 20 }} />}
              {weaponName || "-"}
            </td>
          </tr>
          <tr>
            <td style={{ padding: "2px 4px", borderBottom: "1px solid var(--color-border)", color: "var(--color-text-muted)" }}>SP</td>
            <td style={{ padding: "2px 4px", borderBottom: "1px solid var(--color-border)", display: "flex", alignItems: "center", gap: 4 }}>
              {specialIcon && <img src={specialIcon} alt="" style={{ width: 20, height: 20 }} />}
              {specialName || "-"}
            </td>
          </tr>
        </Fragment>
      );
    })}
  </table>
)}
```

## 4. レイアウト崩れの修正

添付画像から確認できる問題:

### 4a. A枠100s付近の湧きフキダシが凡例と重なっている
100s付近の湧きマーカーのフキダシ（「100.0s 右(1) 4P」）がB枠側の凡例エリアに重なって見えにくい。

対策: 凡例の配置Y座標を下げる（B枠灰色エリアの中央付近に配置）。
または凡例に白背景を付けて重なった場合でも読めるようにする（zIndexで凡例を上にする）。

### 4b. 方面別統計テーブルの#2行目の方面名が欠落
「2 左 3 3」のはずが「2」の後に方面名がないように見える（画像の解像度の問題かもしれない）。
→ テーブルのセル幅を十分に確保し、テキストが省略されないようにする。

### 4c. 右ペインの詳細テキストの折り返し
ブキ名が長くて折り返しが発生し、レイアウトが崩れている。
→ 3. のテーブル形式化で解決される。

### 4d. フッターの生成日時が右下に寄りすぎ
生成日時をフッター行の右端ではなく中央または右寄せで統一的に配置する。

### 修正箇所: `src/components/ImageExport/ExportRenderer.tsx`

各問題に対応する修正をExportRenderer内で行う。

## 完了条件
- 画像出力ボタン押下後にボタンが非活性になり、完了後「画像出力完了」フキダシが表示されること
- フキダシが2秒後に自動消失すること
- 画像内のキケン度が右ペイン最上部に表示されること
- ヘッダー行にキケン度が表示されないこと
- ブキ/SPが表形式（1P〜4Pの行、ブキ/SP列、アイコン+名前列）で表示されること
- 4人とも未設定の場合はブキ/SPテーブルが非表示になること
- 1人でも設定されていれば未設定は「-」表示でテーブルが表示されること
- A枠100s付近のフキダシと凡例が重ならないこと（または重なっても双方読めること）
- 方面別統計テーブルのセルが適切な幅で表示されること
- 生成日時の配置が適切であること
