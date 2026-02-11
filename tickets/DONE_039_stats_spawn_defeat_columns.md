# 039: 方面別統計に「湧き」と「撃破数」の2列を表示し、設定パネルの"方面|湧き|撃破"サマリーを削除する

## 種別
UI改善

## 問題
1. 設定パネルの `方面: 7 | 湧き: 3 | 撃破: 3` というサマリーは方面別統計で確認できるため不要。
2. 方面別統計テーブルの「数」列は湧き数だが、撃破数も見たい。

## 撃破数の分類基準（重要）
撃破は「その撃破から生まれるリスポーンの方面」ではなく、**「撃破されたグリルがどの湧き点から生まれたか」という出自**で分類する。

例: 100秒時点で「正面」から湧いたA枠グリルを95秒で撃破した場合、この撃破は「正面」の撃破として計上する（撃破時刻やリスポーン先の方面は関係ない）。

つまり、湧き→撃破のチェーン対応で、撃破に対応する湧き点の方面区間に撃破をカウントする。

## 修正箇所
- `src/types/computed.ts` — DirectionStats に defeatCount を追加
- `src/utils/statistics.ts` — 撃破数の集計を追加
- `src/components/Statistics/DirectionStatsTable.tsx` — テーブル列を変更
- `src/ScenarioView.tsx` — サマリー表示の削除
- `src/hooks/useGrillCalculation.ts` — directionStats の計算に defeats を渡す（既に渡している場合は変更不要）

## 修正内容

### 型の変更

```ts
// src/types/computed.ts
export interface DirectionStats {
  readonly directionIndex: number;
  readonly direction: string;
  readonly spawnCount: number;   // 旧 count → spawnCount にリネーム
  readonly defeatCount: number;  // 新規追加
}
```

### 統計計算の修正

撃破数の集計ロジック:

```ts
// src/utils/statistics.ts

// 撃破数集計: 湧き→撃破のチェーン対応で、撃破に対応する「湧き点」の方面区間にカウント
const defeatCounts = new Array<number>(sortedDirections.length).fill(0);

for (const defeat of defeats) {
  // この撃破に対応する湧き点を探す（同枠・撃破時刻以上で最も近い湧き）
  const matchingSpawn = spawns
    .filter((s) => s.slot === defeat.slot && s.frameTime >= defeat.frameTime)
    .sort((a, b) => a.frameTime - b.frameTime)[0]; // frameTime が最小（最も近い）

  if (matchingSpawn) {
    // 湧き点のスポナー決定時刻で方面区間を判定
    const decisionFrame = getSpawnerDecisionFrame(matchingSpawn, defeats);
    const index = findDirectionIndex(decisionFrame, sortedDirections);
    if (index >= 0 && index < defeatCounts.length) {
      defeatCounts[index]!++;
    }
  }
}
```

ポイント: `matchingSpawn` は「この撃破で倒されたグリルの湧き点」。湧き→撃破チェーン（006で実装済み）と同じロジックで、同枠内で撃破時刻以上の湧き点のうち最も近いものが対応する湧き点。

ただし、複数の撃破が同じ湧き点にマッチしないよう注意が必要。006のチェーン検証と同様に、降順で1対1マッチングを行うべき:

```ts
// 枠ごとに降順ソートした湧きと撃破を1対1マッチング
for (const slot of ['A', 'B'] as const) {
  const slotSpawns = spawns
    .filter((s) => s.slot === slot)
    .sort((a, b) => b.frameTime - a.frameTime);
  const slotDefeats = defeats
    .filter((d) => d.slot === slot)
    .sort((a, b) => b.frameTime - a.frameTime);

  let spawnIdx = 0;
  for (const defeat of slotDefeats) {
    // spawnIdx の湧きが撃破に対応（チェーン順序で1対1）
    while (
      spawnIdx < slotSpawns.length &&
      slotSpawns[spawnIdx]!.frameTime < defeat.frameTime
    ) {
      spawnIdx++;
    }
    if (spawnIdx >= slotSpawns.length) break;

    const matchedSpawn = slotSpawns[spawnIdx]!;
    const decisionFrame = getSpawnerDecisionFrame(matchedSpawn, defeats);
    const dirIndex = findDirectionIndex(decisionFrame, sortedDirections);
    if (dirIndex >= 0 && dirIndex < defeatCounts.length) {
      defeatCounts[dirIndex]!++;
    }
    spawnIdx++; // この湧きは消費済み
  }
}
```

### テーブルの列変更

```
# | 方面 | 湧き | 撃破
```

- 「数」→「湧き」にリネーム
- 「撃破」列を追加
- 合計行にも両方の合計を表示
- 同名方面集約にも湧き・撃破両方を表示

### サマリーの削除

ScenarioView の設定パネルから以下を削除:
```tsx
// 削除
<div className="text-xs text-text-muted">
  方面: {state.directions.length} | 湧き: {spawns.length} | 撃破: {totalGrillCount}
</div>
```

## 完了条件
- 方面別統計テーブルに「湧き」と「撃破」の2列が表示されること
- 撃破数が「撃破されたグリルの出自（湧き点の方面）」で分類されていること
- 合計行に湧き合計と撃破合計が表示されること
- 同名方面集約にも両方の値が表示されること
- 設定パネルの「方面: N | 湧き: N | 撃破: N」サマリーが削除されていること
