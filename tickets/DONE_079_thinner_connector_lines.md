# 079: リスポーン接続線の線幅を細くする

## 種別
UI改善

## 修正箇所
- `src/components/Timeline/RespawnConnector.tsx`

## 修正内容

撃破マーカーから湧きマーカーへの接続線（実線・破線とも）の `strokeWidth` を細くする。

現在:
```ts
const strokeW = Math.max(2 * scaleX, 1);
```

変更後:
```ts
const strokeW = Math.max(1 * scaleX, 0.5);
```

スポナー決定マークの circle の strokeWidth も合わせて調整:

現在:
```ts
strokeWidth={Math.max(1 * scaleX, 0.5)}
```

変更後:
```ts
strokeWidth={Math.max(0.5 * scaleX, 0.5)}
```

## 完了条件
- 撃破→スポナー決定の実線が 1px 相当の細さになること（ズーム100%時）
- スポナー決定→湧きの破線が 1px 相当の細さになること（ズーム100%時）
- ズーム50%時でも線が見えること（最低0.5px）
