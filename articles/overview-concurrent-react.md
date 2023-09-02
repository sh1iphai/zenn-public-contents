---
title: "【React 18】Concurrent features概要"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "nextjs", "typescript"]
published: false
---

## Concurrent featuresとは

Concurrentが日本語訳で「同時」という意味であることからも推測できるように、Concurrent featuresとは「画面上にUIを表示させつつ、裏でレンダリングの準備を行う」といった並行処理を実現できるようにしたReactの新しいメカニズムです。
Reactの公式ドキュメントでは、Concurrent featuresを搭載したReactのことをConcurrent Reactと呼んでいます。[^1]

[^1]: [What is Concurrent React? ](https://react.dev/blog/2022/03/29/react-v18#what-is-concurrent-react)

Concurrent featuresの最大の特性はレンダリングを中断できるということです。

従来のReactではレンダリングが行われた場合はレンダリングが完了するまで新しい画面や次の操作を行えず、待ち時間が発生します。
一方、Concurrent featuresではレンダリングを並行処理できるため、従来であれば必要であった待ち時間を削減することができ、ユーザーエクスペリエンスの向上が期待できます。

Concurrent featuresは従来のReactのコンセプトを変える仕組みとなっているため、Concurrent featuresがReact 18における最大のアップデートいっても過言ではありません。

## React 18とConcurrent featuresについて

React 18で導入された機能の一覧は以下の通りです。

- Automatic Batching（自動バッチ）
- Transitions
- SuspenseのSSR対応
- React DOM ClientのAPI追加
  - createRoot
  - hydrateRoot
- React DOM ServerのAPIの追加
  - renderToPipeableStream
  - renderToReadableStream
- Strictモードの挙動の変更
- 新しいHooksの追加
  - useId
  - useTransition
  - useDeferredValue
  - useSyncExternalStore
  - useInsertionEffect

上記のうち、Concurrent featuresと密接に関わりのある機能は「Transitions」「SuspenseのSSR対応」「useTransition」「useDeferredValue」です。

以下では、Conccurent featuresが関係するReact 18の新機能の概要について紹介します。

## SuspenseのSSR対応

Suspense自体はReact 16.6で導入されていましたが、React.lazyと組み合わせたコード分割が主な利用方法でした。
React 18からはSuspenseがサーバー上で利用できるようになりました。

SuspenseのSSR対応について話をする前に、まずはSuspenseとSSRについてそれぞれおさらいをします。

### 復習: Suspenseとは
Suspenseとはコンポーネントのレンダリングが完了するまでの間、フォールバックと呼ばれる代替画面を表示する機能を提供するものです。

Suspenseの利用手順は以下の通りです。

1. 対象のコンポーネントをSuspenseで囲う
1. fallbackを指定する

たとえば、fallback中に`Spinner`というコンポーネントを表示するSuspenseを`Cooment`というコンポーネントに適用する場合は以下のようなコードになります。

```jsx
<Suspense fallback={<Spinner />}>
  <Comments />
</Suspense>
```

なお、SuspenseはConcurrent featureにおいて重要な役割を果たしますが、Suspense自体はReact16.6からすでに存在しているため、SuspenseがReact 18の新機能というわけではありません。

Suspenseの活用パターンとしては主に、コード分割と呼ばれる遅延ローディングを実装する場合や、API経由でのデータ取得をはじめとした非同期処理中のローディングを表示する場合などがあります。

#### Suspenseによるコード分割

コード分割とは、画面の初期表示に不要なファイルを遅延読み込みすることで、初期ロードの時間を減らし画面の表示速度を向上させるものです。

コード分割はReact.lazyとSuspenseを組み合わせることで実現できます。

React.lazyを利用してimportされたコンポーネントは遅延読み込みされ、コンポーネントの読み込み中はSuspenseで定義したfallbackが表示されます。

具体的なコードはたとえば以下のようになります。

```jsx
import { Suspense, lazy } from "react";
const HeavyMessage = lazy(() => import("./HeavyMessage"));

export const App = () => {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <HeavyMessage />
    </Suspense>
  );
};

```

React 18以前では、Suspenseは主にこのケース、つまりコード分割をするために利用されていました。


#### Suspenseと非同期処理の組み合わせ

Suspenseは非同期処理と組み合わせることで、非同期処理中はSuspenseのフォールバックを表示し、非同期処理が完了したらコンポーネントを表示するといったことが実現できます。
いわゆるAPI経由で非同期にデータを処理する場合がこの例にあたります。

非同期処理の状態に応じて表示を制御できるのはSuspenseの特性によるものです。

Suspenseの特性をまとめると以下のようになります。

- Suspense内でthrowされたPromiseをcatchする
- Promiseがpendingの時はフォールバックをレンダリングする
- Promiseがsettled（fulfilledあるいはrejected）の時はコンポーネントをレンダリングする

なお、Promiseとはpending（保留）、fulfilled（成功）、rejected（失敗）の3つの状態を持つ非同期処理を表すオブジェクトです。


Suspense内で非同期処理が開始されてから完了するまでの流れのイメージは以下の通りです。

- Suspenseがコンポーネントのレンダリングを試みる
- コンポーネントからPromiseがthrowされる
- SuspenseがthrowされたPromiseをcatchし、Promiseの処理を行う
- Promiseを処理中（pending）の場合、fallbackをレンダリングする
- Promiseの処理が完了（fulfilled or rejected）したら再度コンポーネントのレンダリングを試みる
- レンダリングがうまくいったらfallbackからコンポーネントに表示が切り替わる

図で表現すると以下のようになります。

まずはSuspenseが、Suspense内のコンポーネントのレンダリングを試みます。

![](/images/overview-concurrent-react/suspense-1.png)

次に、非同期処理が組み込まれているコンポーネントからPromiseがthrowされます。

![](/images/overview-concurrent-react/suspense-2.png)

PromiseがthrowされるとSuspenseはそのPromiseをcatchし

![](/images/overview-concurrent-react/suspense-3.png)


Promiseが処理中、つまりpendingの場合はSuspenseのフォールバックをレンダリングします。

![](/images/overview-concurrent-react/suspense-4.png)

Promiseの処理が終わり、Promiseの状態が完了、つまりfulfilledもしくはrejectedになったら再度コンポーネントのレンダリングを試みます。

![](/images/overview-concurrent-react/suspense-5.png)

レンダリングがうまくいったらフォールバックの代わりにコンポーネントに表示を切り替えます。

![](/images/overview-concurrent-react/suspense-6.png)

## 復習: SSRとは
SSRとはサーバー側でHTMLを生成し、その結果をクライアントに送信する仕組みのことをいいます。

SSRの大まかな流れは以下の通りです。

- データの取得
- HTMLを返す
- JavaScriptをロードする
- HTMLとJavaScriptのつなぎこむ

図で表現すると以下のようになります。

サーバにリクエストが届くとまずはSSRに必要なデータの取得を行います。

![](/images/overview-concurrent-react/ssr-1.png)

必要なデータがそろったら、サーバー側でHTMLを生成し、クライアント側へレスポンスとして返します。

![](/images/overview-concurrent-react/ssr-2.png)

HTMLを受け取ったクライアントは、JavaScriptのロードを実行します。

![](/images/overview-concurrent-react/ssr-3.png)

HTMLとJavaScriptの準備が整ったら、次に、HTMLとJavaScriptのつなぎこみを行います。
具体的にはHTMLに対してイベントリスナーや状態管理などのJavaScriptの機能を付与します。
なお、この過程のことをハイドレーションと呼びます。

ハイドレーションが完了すると、画面はインタラクティブ、つまり操作可能な状態になります。

![](/images/overview-concurrent-react/ssr-4.png)


SSRではクライアント側でHTMLの生成をする必要がなくなるため、初期表示の高速化が期待できます。
また、クローラーに対してレンダリングしたページを見せられるため、SEO対策として有効です。


一方で、SSRには全ての準備が整わないと初期表示ができないという問題点があります。
具体的には、データ取得やJavaScriptのロードやハイドレーションに時間を要するコンポーネントが一部でも存在していれば、それが画面全体の初期表示までにかかる時間に影響を与えます。

### Streaming Server Rendering
SuspenseのSSR対応の話に戻ります。

SuspenseのSSR対応とは、非同期でSSRが実行できるようなったということを意味します。

非同期でSSRの処理が行えるようになった結果、一部で処理の重いコンポーネントがあったとしても、画面全体がその影響を受けることはなくなります。
なお、Suspenseを活用し、非同期でSSRするメカニズムのことをStreaming Server Renderingといいます。

一般的なSSRとStreaming Server Renderingの違いを図で表現すると以下のようになります。
以下の図は、初期画面が表示されるまでの過程を表現したものです。そして、右下に重い処理をする必要があるコンポーネントが配置されている画面とします。

![](/images/overview-concurrent-react/streaming.png)

上記の図の意味について補足説明をします。

SSRの場合は全てのコンポーネントに対して同期的に処理を実行します。
そのため、右下のコンポーネントの処理速度が画面全体に影響を与えていることがわかります。

一方、Streaming Server Renderingの場合は、非同期で処理を行うため、右下のコンポーネントの処理速度は画面全体に影響を与えていないことがわかります。
Streaming Server Renderingでは非同期で処理を行えるため、一部のコンポーネントだけ先んじて画面に表示されていることがわかります。

なお、Streaming Server Renderingによって非同期でサーバーから返されるHTMLはStreming HTMLなどと呼ばれています。

## Transitions
Transitionsは直訳すると「遷移」という意味があります。
TransitionsとはReactのデータ更新に対して「優先度」の観点を追加した新しい概念です。

Transitionsが適用されたデータ更新は優先度が低いものとみなされます。
そのため、Transitionsを活用することで相対的に優先度の高いデータ更新と優先度の低いデータ更新の2種類を用意できます。

優先度の高いデータ更新には、たとえば、クリックや入力といった操作内容を即座に画面に反映させたいユーザーエクスペリエンスに関係するものが分類されます。
一方、優先度が低いデータ更新、つまりTransitionsを適用するデータ更新には、更新結果を即時画面に表示する必要がなく遅延が許されるものが分類されます。

優先度が低いデータ更新によって発生するレンダリングは、優先度が高いデータ更新が行われることで中断される、という特徴があります。
その結果、「画面に結果を表示させつつ、裏でレンダリングを進める」といった、並行処理が実現できます。

なお補足ですが、Reactの公式ドキュメントではTransitionsのデータ更新を説明する際に「Urgent updates」と「Non-urgent updates（Transition updates）」という言葉を使っています。[^1]
Urgentは直訳すると「緊急」という意味ですので、「緊急度の高い更新」というような表現を使うほうが適切かもしれませんが、日本語のわかりやすさの観点から、本講座では「優先」という言葉を利用してTransitionsについて説明をしています。

[^1]: [New Feature: Transitions](https://react.dev/blog/2022/03/29/react-v18#new-feature-transitions)

React 18ではTransitions（トランジション）を実現するためのHooksが新たに2つ用意されました。
それが、useTransitionとuseDeferredValueです。

## useTransactionとは

useTransitionとは、トランジションを実現するためのフックの1つで、ステートの更新に対してトランジションを適用できます。

useTransitionの戻り値はisPendingというbooleanとstartTransitionという関数の2つです。
isPendingはTransitionが保留中かどうかを判定するbooleanです。
startTransitionは処理に対してTransitionを適用する関数です。

useTransitionの使い方は以下の通りです。

1. useTransitionの戻り値(isPending, startTransition)を取得
1. startTransitionでTransitionsを適用する

具体例を紹介します。

例えばボタンのクリックによってStateを更新する処理があったとします。

```jsx
const [count, setCount] = useState(0);

const handleClick = () => {
  setCount((prev) => prev + 1);
};

```

上記のState更新に対してuseTransitinoを利用してTransitionを適用する場合は以下のようになります。

```jsx
const [count, setCount] = useState(0);
const [_, startTransition] = useTransition();

const handleClick = () => {
  startTransition(() => {
    setCount((prev) => prev + 1);
  });
};

```


## useDefferedValueとは

useDeferredValueとは、トランジションを実現するためのフックの1つで、値に対してトランジションを適用できます。
useDeferredValueは引数valueに対してトランジションを適用させる関数となっています。

useDifferedValueの使い方は以下の通りです。

1. Transitionsを適用したい値をuseDeferredValueの引数にする
1. useDeferredValueの戻り値を利用する


具体例を紹介します。

例えば、以下のようなqueryというローカルStateを持つコンポーネントについて考えてみます。
このqueryに対してトランジションを適用したい場合は、useDeferredValueの引数にqueryをセットします。
そして、useDeferredValueの戻り値を例えばdeferredQueryといったものにしてあげた場合、deferredQueryを利用することでコンポーネント内でトランジションが実現できます。

```jsx
import { useState, useDeferredValue } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  // ...
}

```
