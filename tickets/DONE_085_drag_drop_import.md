# 085: タイムラインへのドラッグ&ドロップでインポート

## 種別
機能追加

## 概要

タイムラインエリアに JSON ファイルをドラッグ&ドロップすることでインポートできるようにする。

## 修正箇所
- `src/utils/fileIO.ts` — ファイル読み込みロジックの分離
- `src/ScenarioView.tsx` — ドロップイベントハンドラ追加
- `src/components/Timeline/index.tsx` — ドロップゾーンUI

## 修正内容

### 1. fileIO.ts — File オブジェクトから直接インポートする関数を追加

現在の `importScenarioFromFile` は内部で `<input type="file">` を生成してファイル選択ダイアログを開く。
ドラッグ&ドロップでは既に File オブジェクトが得られるため、File を受け取る関数を分離する。

```ts
/**
 * File オブジェクトからシナリオをインポートする
 * （ドラッグ&ドロップ、または直接ファイル参照時に使用）
 */
export function importScenarioFromFileObject(
  file: File,
  hazardConfigData: HazardConfigData,
  weapons: readonly WeaponMaster[],
  specials: readonly SpecialMaster[],
): Promise<ImportResult> {
  return new Promise((resolve) => {
    if (!file.name.endsWith(".json")) {
      resolve({ success: false, error: "JSONファイルのみ対応しています" });
      return;
    }

    const reader = new FileReader();
    reader.onload = () => {
      const text = reader.result;
      if (typeof text !== "string") {
        resolve({ success: false, error: "ファイルの読み込みに失敗しました" });
        return;
      }
      resolve(parseAndValidate(text, hazardConfigData, weapons, specials));
    };
    reader.onerror = () => {
      resolve({ success: false, error: "ファイルの読み込みに失敗しました" });
    };
    reader.readAsText(file);
  });
}
```

既存の `importScenarioFromFile` は内部で `importScenarioFromFileObject` を呼ぶようリファクタリング:

```ts
export function importScenarioFromFile(
  hazardConfigData: HazardConfigData,
  weapons: readonly WeaponMaster[],
  specials: readonly SpecialMaster[],
): Promise<ImportResult> {
  return new Promise((resolve) => {
    const input = document.createElement("input");
    input.type = "file";
    input.accept = ".json";

    input.onchange = async () => {
      const file = input.files?.[0];
      if (!file) {
        resolve({ success: false, error: "ファイルが選択されませんでした" });
        return;
      }
      resolve(await importScenarioFromFileObject(file, hazardConfigData, weapons, specials));
    };

    input.click();
  });
}
```

`parseAndValidate` は既存のまま（内部関数として維持）。

### 2. ScenarioView.tsx — ドロップハンドラ

```tsx
const handleFileDrop = useCallback(
  async (file: File) => {
    const result = await importScenarioFromFileObject(file, hazardConfigData, weapons, specials);
    if (result.success && result.scenario) {
      dispatch({ type: "LOAD_SCENARIO", payload: result.scenario });
      setIoError(null);
      if (result.warnings && result.warnings.length > 0) {
        setIoWarnings(result.warnings);
        setTimeout(() => setIoWarnings([]), 5000);
      } else {
        setIoWarnings([]);
      }
    } else {
      setIoError(result.error ?? "インポートに失敗しました");
      setTimeout(() => setIoError(null), 3000);
    }
  },
  [dispatch, hazardConfigData, weapons, specials],
);
```

`onFileDrop={handleFileDrop}` を Timeline に props で渡す。

### 3. Timeline/index.tsx — ドロップゾーン

```tsx
interface TimelineProps {
  ...
  onFileDrop?: (file: File) => void;  // 追加
}

export function Timeline({ ..., onFileDrop }: TimelineProps) {
  const [isDragOver, setIsDragOver] = useState(false);

  const handleDragOver = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    e.stopPropagation();
    // ファイルのドラッグのみ受け付ける
    if (e.dataTransfer.types.includes("Files")) {
      setIsDragOver(true);
    }
  }, []);

  const handleDragLeave = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    e.stopPropagation();
    setIsDragOver(false);
  }, []);

  const handleDrop = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    e.stopPropagation();
    setIsDragOver(false);

    const file = e.dataTransfer.files[0];
    if (file) {
      onFileDrop?.(file);
    }
  }, [onFileDrop]);

  return (
    <div
      className="rounded-sm border border-border bg-surface"
      onDragOver={handleDragOver}
      onDragLeave={handleDragLeave}
      onDrop={handleDrop}
      style={{
        position: "relative",
      }}
    >
      {/* ドラッグオーバー時のオーバーレイ */}
      {isDragOver && (
        <div
          className="pointer-events-none absolute inset-0 z-50 flex items-center justify-center rounded-sm"
          style={{
            backgroundColor: "rgba(59, 130, 246, 0.1)",
            border: "2px dashed var(--color-primary)",
          }}
        >
          <span className="rounded-md bg-surface px-4 py-2 text-sm font-medium text-primary shadow-sm">
            ドロップしてインポート
          </span>
        </div>
      )}

      {/* 既存のタイムライン内容 */}
      <div className="flex" style={{ ... }}>
        ...
      </div>
    </div>
  );
}
```

### ドラッグオーバー時の視覚効果

- タイムライン全体に薄い青色のオーバーレイ（`rgba(59, 130, 246, 0.1)`）
- 青い破線ボーダー（`2px dashed var(--color-primary)`）
- 中央に「ドロップしてインポート」テキスト
- `pointer-events: none` でオーバーレイ自体はドロップイベントを妨げない

## 完了条件
- タイムラインエリアに JSON ファイルをドラッグすると視覚的フィードバック（オーバーレイ）が表示されること
- ドロップするとインポートが実行されること
- インポート成功時にシナリオが読み込まれること
- インポート失敗時にエラーメッセージが表示されること（既存のエラー表示と同じ）
- JSON以外のファイルをドロップした場合にエラーメッセージが表示されること
- 既存のファイル選択ダイアログによるインポートも引き続き動作すること
- ドラッグしたファイルをタイムライン外に移動した場合にオーバーレイが消えること
