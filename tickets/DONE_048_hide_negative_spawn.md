# 048: 0秒以下に湧く場合のマーカー・接続線を非表示にする

## 種別
バグ（表示）

## 問題
残り数秒時点に撃破マーカーを置くと、リスポーン湧き（撃破から3.57秒後）が0秒以下になり、タイムライン範囲外にマーカーが描画されてしまう。

## 修正箇所
- `src/components/Timeline/GrillSlotLane.tsx` — 湧きマーカー・リスポーン接続線の表示条件

## 修正内容

### 湧きマーカーの非表示条件

frameTime が 0 以下（= 0秒以下）の湧き点は描画しない:

```tsx
{slotSpawns
  .filter((spawn) => spawn.frameTime > 0)
  .map((spawn) => (
    <SpawnMarker key={spawn.id} spawn={spawn} ... />
  ))}
```

### リスポーン接続線の非表示条件

同様に、対応する湧き点の frameTime が 0 以下の場合は RespawnConnector も描画しない:

```tsx
{respawnSpawns
  .filter((spawn) => spawn.frameTime > 0)
  .map((spawn) => {
    const defeat = defeats.find((d) => d.id === spawn.defeatId);
    if (!defeat) return null;
    return <RespawnConnector key={...} ... />;
  })}
```

### 活動期間バーの非表示条件

湧き点の frameTime が 0 以下の場合、対応する ActivePeriod も描画しない。

### 撃破マーカーはそのまま表示

撃破マーカー自体は撃破が行われた時刻に表示する（0秒以上であれば）。湧き/接続線のみ非表示にすることで「撃破はしたがリスポーンはウェーブ終了で発生しなかった」ことが視覚的にわかる。

### ドラッグ中の動的対応

ドラッグで撃破マーカーを移動した結果、リスポーン湧きが0秒以上になった場合は通常通り表示する。これは spawns が撃破位置から再計算されるため、自動的に対応する。

## 完了条件
- 湧き frameTime ≤ 0 の場合、湧きマーカーが表示されないこと
- 同条件でスポナー決定マーカー・リスポーン接続線も表示されないこと
- 撃破マーカーは引き続き表示されること
- ドラッグで撃破位置を変更し湧きが 0 秒以上になったら表示されること
