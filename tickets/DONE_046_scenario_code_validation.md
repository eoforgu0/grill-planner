# 046: シナリオコード入力のバリデーションとフォーマット自動整形

## 種別
UI改善（機能追加）

## シナリオコード仕様
- 形式: `Sxxx-xxxx-xxxx-xxxx`（先頭S + 16桁の英数、4桁ハイフン区切り）
- 有効文字: `ABCDEFGHJKLMNPQRSTUVWXY0123456789`（半角大文字から I, O, Z, W を除外…ではなく、I, O, Z を除外。W は含む）
  - 正確な有効文字セット: `A B C D E F G H J K L M N P Q R S T U V W X Y 0 1 2 3 4 5 6 7 8 9`（33文字）
- 先頭は必ず `S` 固定
- ハイフンを除くと17文字（S + 16桁）

## バリデーション仕様

### 自動変換（入力時・ペースト時）
1. 全角英数 → 半角に変換（例: `Ｓ０１２` → `S012`）
2. 小文字 → 大文字に変換（例: `sabc` → `SABC`）
3. 有効文字以外を除去（ハイフン以外の記号、I, O, Z 等）
4. ハイフンの自動挿入: 1文字目の後に4文字ごとにハイフンを挿入
   - 入力中のハイフンはユーザーが打っても打たなくてもよい
   - システムが正規化して常に `Sxxx-xxxx-xxxx-xxxx` 形式にする

### バリデーション判定
- **空**: バリデーションしない（エラーにしない）
- **入力中（17文字未満）**: エラー表示しない（まだ入力途中）
- **17文字（ハイフン除く）以上**: 以下をチェック
  - 先頭が `S` であること
  - 全文字が有効文字であること
  - 長さが正確に17文字（S + 16桁）であること
- **バリデーションエラー**: input の border を赤くする

### ペースト対応
- ペーストされた文字列にも全変換ルールを適用
- ペースト元にハイフンや空白が含まれていても正しく処理

## 修正箇所
- `src/components/Settings/MemoSection.tsx` — シナリオコード入力の onChange/onPaste ロジック
- （必要なら新規ユーティリティ `src/utils/scenarioCode.ts`）

## 修正内容

### 正規化関数

```ts
const VALID_CHARS = new Set('ABCDEFGHJKLMNPQRSTUVWXY0123456789'.split(''));

function normalizeScenarioCode(raw: string): string {
  // 1. 全角→半角
  let s = raw.replace(/[Ａ-Ｚａ-ｚ０-９]/g, (ch) =>
    String.fromCharCode(ch.charCodeAt(0) - 0xFEE0),
  );
  // 2. 小文字→大文字
  s = s.toUpperCase();
  // 3. ハイフンと有効文字以外を除去
  s = s.replace(/[^ABCDEFGHJKLMNPQRSTUVWXY0123456789-]/g, '');
  // 4. ハイフンを全除去（再挿入するため）
  s = s.replace(/-/g, '');
  // 5. 最大17文字（S + 16桁）に切り詰め
  s = s.slice(0, 17);
  // 6. ハイフン自動挿入: S + xxxx-xxxx-xxxx-xxxx
  if (s.length > 1) {
    const head = s[0]; // 'S'
    const rest = s.slice(1);
    const parts = rest.match(/.{1,4}/g) ?? [];
    s = head + parts.join('-');
  }
  return s;
}
```

### バリデーション関数

```ts
function validateScenarioCode(normalized: string): boolean {
  if (normalized === '') return true; // 空はOK
  // ハイフンを除去して検証
  const bare = normalized.replace(/-/g, '');
  if (bare.length < 17) return true; // 入力途中はOK
  if (bare.length !== 17) return false;
  if (bare[0] !== 'S') return false;
  return [...bare].every((ch) => VALID_CHARS.has(ch));
}
```

### 入力ハンドラ

```tsx
const [codeError, setCodeError] = useState(false);

const handleCodeChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  const normalized = normalizeScenarioCode(e.target.value);
  onSetScenarioCode(normalized);
  setCodeError(!validateScenarioCode(normalized));
};
```

### UI

```tsx
<input
  type="text"
  value={memo.scenarioCode}
  onChange={handleCodeChange}
  placeholder="Sxxx-xxxx-xxxx-xxxx"
  maxLength={20}  // 17文字 + ハイフン3本 = 20
  className={`w-52 rounded-sm border px-2 py-1 text-sm text-text ${
    codeError ? 'border-danger' : 'border-border'
  } bg-surface`}
/>
```

### カーソル位置の考慮

ハイフン自動挿入でカーソル位置がずれる問題がある。正規化後にカーソル位置を補正する:

```tsx
const inputRef = useRef<HTMLInputElement>(null);

const handleCodeChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  const el = e.target;
  const cursorBefore = el.selectionStart ?? 0;
  const rawBefore = el.value;

  const normalized = normalizeScenarioCode(el.value);
  onSetScenarioCode(normalized);
  setCodeError(!validateScenarioCode(normalized));

  // ハイフンを除いた実質文字数でカーソル位置を計算
  const barePosBefore = rawBefore.slice(0, cursorBefore).replace(/-/g, '').length;
  // normalized のうち barePosBefore 番目の実質文字の後ろにカーソルを置く
  let bareCount = 0;
  let newCursor = 0;
  for (let i = 0; i < normalized.length; i++) {
    if (normalized[i] !== '-') bareCount++;
    if (bareCount >= barePosBefore) {
      newCursor = i + 1;
      break;
    }
  }
  if (bareCount < barePosBefore) newCursor = normalized.length;

  requestAnimationFrame(() => {
    el.setSelectionRange(newCursor, newCursor);
  });
};
```

## 完了条件
- 全角英数が自動で半角に変換されること
- 小文字が自動で大文字に変換されること
- ハイフンが4桁ごとに自動挿入されること
- I, O, Z などの無効文字が入力・ペーストされても除去されること
- 空の場合はエラーにならないこと
- 17文字（ハイフン除く）入力後、形式が正しくなければ input の枠が赤くなること
- 形式が正しければ赤くならないこと
- ペースト時にも全変換が正しく動作すること
- プレースホルダが「Sxxx-xxxx-xxxx-xxxx」であること
