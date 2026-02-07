---
layout: post
title: "React 18/19 変更点まとめ: ref / forwardRef / useRef のTypeScriptパターン"
date: 2026-01-20 0:00:00 +900
comments: true
tags: ["react", "typescript"]
categories: ["Blog", "フロントエンド基礎"]
image: /assets/img/2026/react-typescript-useref-dom-ref-mutable-ref.png
---

## 背景

しばらく React 17 くらいの感覚で開発していて、久々に「今どきの React の書き方（特に TypeScript 周り）」をまとめてキャッチアップしました。

今回のキャッチアップ対象はここ:

[typescript-cheatsheets/react（React + TypeScript Cheatsheets）](https://github.com/typescript-cheatsheets/react)

## React 18/19 変更点まとめ

### React 18: React.FC（FunctionComponent）を“必ずしも使わない”が主流に

昔の TypeScript React だと `React.FC` をよく見ましたが、今は「必須ではない」「環境によっては非推奨」という整理。

さらに、React 18 type 以前は `React.FC` が `children` を暗黙に含んでいた点が議論になった…という背景も書かれています。

関連で、過渡期の `React.VFC` / `VoidFunctionComponent` は React 18 で deprecated になっている。

自分はこの書き方が一番好き。

```tsx
// Easiest way to declare a Function Component; return type is inferred.
const App = ({ message }: AppProps) => <div>{message}</div>;
```

### React 18:createRoot 前提で Automatic Batching が効く

React 18 の目立つ変化の一つが Automatic Batching（自動バッチング）。React 17 だと「Reactのイベントハンドラ内はまとめて再描画されるけど、`setTimeout` / `Promise` の中は別」みたいな感覚があった。

React 18 では、`Promise` や `setTimeout`、ネイティブイベント等の中で起きた state 更新も、まとめて1回の再レンダリングにされる（= `automatic batching`）。

```tsx
setTimeout(() => {
  setCount((c) => c + 1);
  setFlag((f) => !f);
}, 1000);
```

この前提が入ると、体感としては

- 余計な再レンダリングが減る（うれしい）
- “更新がすぐ反映される前提” の雑なコードは、タイミング差で違和感が出ることがある（注意）

みたいな方向に寄る。


### React 19:ref が props として扱える（forwardRef の温度感が変わる）

React 19 で個人的にわかりやすく「書き方が変わった」と感じたのがここ。

function component が `ref` を `props` として受け取れるようになった。

```tsx
function MyInput({ placeholder, ref }: { placeholder?: string; ref?: React.Ref<HTMLInputElement> }) {
  return <input placeholder={placeholder} ref={ref} />;
}
```

React 側も「新しい function component では `forwardRef` が不要になる」方向を明言していて、将来的に `forwardRef` は [**deprecated** 予定](https://ja.react.dev/blog/2024/12/05/react-19#ref-as-a-prop)、という流れ。

### React 19:Actions は“まだ試せてない”ので、メモだけ

React 19 では `Actions` が入って、フォーム送信の `pending` / `error` / 状態遷移まわりの扱いが厚くなってる。
ただ、自分はまだ手元のコードベースでちゃんと試せてないので、ここは「追々触る」枠として残しておく。

### 追加:useEffect の “戻り値” の落とし穴

`useEffect` は `cleanup` 関数か `undefined` 以外を返すと怒られるので、アロー関数の書き方次第で `setTimeout` の戻り値（number）を返してしまう例が載ってます。

→ “昔のノリで短く書いて事故る” 代表格。

```tsx
function DelayedEffect(props: { timerMs: number }) {
  const { timerMs } = props;

  useEffect(
    () =>
      setTimeout(() => {
        /* do stuff */
      }, timerMs),
    [timerMs]
  );
  // bad example! setTimeout implicitly returns a number
  // because the arrow function body isn't wrapped in curly braces
  return null;
}
```

これは **React 18→19 の変更というより「Hooks（useEffect）が登場した最初からのルール」**だったようで、最近のTypeScriptの設定の厳格化やStrictModeの登場などで検出されやすくなっただけのよう。


### 追加:useRef は「DOM参照」か「可変箱」かで型付けが分かれる

キャッチアップして一番「へー」と思ったのがこれ。`useRef` は用途が2種類あって、型と初期値の置き方が変わる。

ざっくり言うとこうです:

- DOM参照:`ref.current` は「React が差し込むもの」なので、初期値は `null` で始めて、後から要素が入る（= `T | null`）。
- 可変箱:自分で `.current` を更新して使う “インスタンス変数” みたいな用途。初期値に `null` を入れると意味が変わる（読み取り専用寄り）ので、型/初期値の置き方が別になります。

#### 1) DOM参照（最も典型）

```tsx
import { useEffect, useRef } from "react";

export function FocusInput() {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    inputRef.current?.focus(); // null の可能性があるので ?. が自然
  }, []);

  return <input ref={inputRef} />;
}
```

ポイント:

- `useRef<HTMLInputElement>(null)` の形（`HTMLElement` より 具体型を推奨）
- `current` は `HTMLInputElement | null` になるので `null` 安全に扱う

#### 2) 可変箱（最新値を保持して、ハンドラ内で参照）


```tsx
import { useEffect, useRef, useState } from "react";

export function LatestValueExample() {
  const [count, setCount] = useState(0);
  const latestCountRef = useRef(count);

  useEffect(() => {
    latestCountRef.current = count;
  }, [count]);

  useEffect(() => {
    const id = window.setInterval(() => {
      // 常に最新の count を参照できる
      console.log("latest:", latestCountRef.current);
    }, 1000);
    return () => window.clearInterval(id);
  }, []);

  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>;
}
```

ポイント:
- “可変箱” は 自分で `.current = ...` と更新するのが前提
- 状態じゃないので `.current` が変わっても再描画しない（= それが狙い）

ただ注意点としては `TypeScript` 的には（型レベルでは）`current` への代入が **“readonly 扱い”** になりやすいだけです。JS 実行時に本当に書き換え不能になるわけではない。

## まとめ

[typescript-cheatsheets/react（React + TypeScript Cheatsheets）](https://github.com/typescript-cheatsheets/react) は、いわゆる「最初から全部を体系的に学ぶ」タイプのドキュメントというより、現場で悩みがちな React + TypeScript の型付けに対して“結論”がまとまっている資料です。ブランク明けに読むには、ちょうど良い距離感でした。

うちのプロジェクトはまだ React 17 のものもあって、どうしても最新のキャッチアップが後回しになりがちです。でも、たまにまとめて追うだけでも、**「普段の実装に直結する地味な差分」**が一気に整理できて、効果が大きいと感じました。