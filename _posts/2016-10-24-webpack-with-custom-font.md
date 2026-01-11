---
layout: post
title: "webpack+Sass+reactでカスタムフォントを使う"
date: 2016-10-24 01:58:00 +900
comments: true
tags: [webpack, react, sass, font]
categories: ["Blog", "フロントエンド基礎"]
image: https://s3-ap-northeast-1.amazonaws.com/blog-mitsuruog/images/2016/custom-font.png
---

アイコンなどの画像には、少し前まで CSS Sprites を使うことが多かったと思います。しかし、最近は CSS でスタイリングできるため、カスタムフォントを作成して使用することが増えてきたように感じます。

今回は、webpack と Sass を使ってカスタムフォントを使用する方法と、その際にハマりやすい`url()`での相対パス指定方法について紹介します。

<!-- more -->

## tl;dr

- Sass でのカスタムフォントの使い方
- webpack でカスタムフォントをロードする方法
- Sass での`url()`での相対パスの指定方法のハマりポイント

## はじめに

はじめに登場人物が多いので、全体のフォルダ構成を明らかにしておきます。
JavaScript のフレームワークは React を使ったのもを想定していますが、カスタムフォント周りの設定については他のフレームワークでも代用できると思います。

```
src/
  components/
    icon/
      icon.scss
      icon.jsx
      My-font.eot
      My-font.woff
      My-font.ttf
      My-font.svg
  entry.js
  style
    style.scss
 webpack.config.js
```

`entry.js`がアプリケーションのエントリーファイルです。
`style.scss`は Sass のエントリーファイルで、`icon.scss`を`@import`しています。
`entry.js`がロードされる時に、`style.scss`もロードされインポートされている Scss ファイルがコンパイルされます。

```scss
// src/style.scss
@import "../components/icon/icon.scss";
```

## Sass でのカスタムフォントの使い方

### Sass ファイルでカスタムフォントを読み込む

カスタムフォントを読み込むには`@font-face`を利用して`font-family`にフォント名を設定します。

カスタムフォントの読み込みについては、Sass ではなく CSS の範疇ですし、詳細については他に良い記事が多くあるのでここでは触れません。

```scss
// src/components/icon/icon.scss
@font-face {
  font-family: "SuperCoolMyFont";
  src: url("./My-font.eot"),
    /* IE9 Compat Modes */ url("./My-font.eot?iefix") format("embedded-opentype"),
    /* IE6-IE8 */ url("./My-font.woff") format("woff"),
    /* Pretty Modern Browsers */ url("./My-font.ttf") format("truetype"), /* Safari, Android, iOS */
      url("./My-font.svg#myfont") format("svg"); /* Legacy iOS */
}
```

### icon コンポーネントの作成

icon コンポーネントは stateless function components で作成した非常にシンプルなものです。
将来的には、アイコンの種類やサイズを prop から渡せるようにするべきだと思います。

```js
// src/components/icon/icon.jsx
exports default function Icon() => {
  return <span className="icon" />
}
```

### icon のスタイリング

icon のスタイリングについては、Bootstrap など既存の CSS フレームワークを参考にするのがいいと思います。
基本的には icon のベースとなるスタイルに`font-family`でフォントを指定した後、`content`でフォントの Unicode を設定すればいいです。

```scss
// src/components/icon/icon.scss
.icon {
  position: relative;
  display: inline-block;

  font-family: "SuperCoolMyFont";
  font-style: normal;
  font-weight: normal;
  font-size: 24px;

  line-height: 1;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;

  &:before {
    content: "\01";
  }
}
```

### フォントの Unicode の探し方(番外編)

フォントの Unicode の確認方法については、自分の場合は SVG ファイルから探すようにしています。
(正直、これがスマートな方法かどうかわかりません。)

SVG ファイルを開くと`unicode`要素があるので、そこから Unicode を抽出します。
例えば、`unicode="&#x01;"`の場合は`&#x`を`\`に変換して`\01`として使います。

```
// src/components/icon/My-font.svg
// 抜粋です
<glyph unicode="&#x01;" d="something cool shape" />
```

もっと正しい方法があったら教えて欲しいです。

## webpack でカスタムフォントをロードする方法

webpack でカスタムフォントを使うためには、[url-loader](https://github.com/webpack/url-loader)を使って各フォントファイルをロードできるようにする必要があります。

```js
// webpack.config.js
// 抜粋です
loaders: [
  {
    test: /\.svg$/,
    loader: "url?mimetype=image/svg+xml&name=[path][name].[ext]",
  },
  {
    test: /\.woff$/,
    loader: "url?mimetype=application/font-woff&name=[path][name].[ext]",
  },
  {
    test: /\.woff2$/,
    loader: "url?mimetype=application/font-woff2&name=[path][name].[ext]",
  },
  {
    test: /\.[ot]tf$/,
    loader: "url?mimetype=application/octet-stream&name=[path][name].[ext]",
  },
  {
    test: /\.eot$/,
    loader:
      "url?mimetype=application/vnd.ms-fontobject&name=[path][name].[ext]",
  },
];
```

`name`の部分は好みで変更してください。

私の場合は SVG ファイルをフォントと画像で利用することが多く、これらを別のフォルダで管理することが多いので、格納元のディレクトリ構造そのままで出力してくれる`[path][name].[ext]`を好んで利用しています。

## Sass での`url()`での相対パスの指定方法のハマりポイント

ところが、上の方法では font ファイルが参照できずエラーとなってしまいます。

```
Module not found: Error: Cannot resolve 'file' or 'directory' . /My-font.svg
```

これは現時点では、Sass のコンパイラが URL の書き換えをサポートしていないため、`css-loader`を利用する場合は Sass のエントリーファイルから相対パス指定する必要があるからです。

- [sass-loader: Problems with url(...)](https://github.com/jtangelder/sass-loader#problems-with-url)

そのため、上の`icon.scss`でのパス指定は次のようにする必要があります。

```scss
// src/components/icon/icon.scss
$font-path: "../components/icon/";
@font-face {
  font-family: "SuperCoolMyFont";
  src: url("#{$font-path}My-font.eot"),
    /* IE9 Compat Modes */ url("#{$font-path}My-font.eot?iefix") format("embedded-opentype"),
    /* IE6-IE8 */ url("#{$font-path}My-font.woff") format("woff"),
    /* Pretty Modern Browsers */ url("#{$font-path}My-font.ttf") format("truetype"),
    /* Safari, Android, iOS */ url("#{$font-path}My-font.svg#myfont") format("svg"); /* Legacy iOS */
}
```

これで、相対パスの問題は解決されてフォントを読み込むことができるました。しかし、相対パスの指定が実物のフォントと異なるので違和感が残ります。

私の場合、`$font-path`を Sass のエントリーファイルに移動することで、その違和感を解消しています。

```scss
// src/style.scss
$font-path: "../components/icon/";
@import "../components/icon/icon.scss";

// src/components/icon/icon.scss
@font-face {
  font-family: "SuperCoolMyFont";
  src: url("#{$font-path}My-font.eot"),
    /* IE9 Compat Modes */ url("#{$font-path}My-font.eot?iefix") format("embedded-opentype"),
    /* IE6-IE8 */ url("#{$font-path}My-font.woff") format("woff"),
    /* Pretty Modern Browsers */ url("#{$font-path}My-font.ttf") format("truetype"),
    /* Safari, Android, iOS */ url("#{$font-path}My-font.svg#myfont") format("svg"); /* Legacy iOS */
}
```

これなら違和感なしです。

## まとめ

webpack+Sass でカスタムフォントを使う方法でした。
Sass の相対パスまわりでハマるケースが多いのではないかと思います。

`url()`に関してはフォントだけではなく画像についても同じことが言えるので、それぞれのエコシステムの癖を把握する必要があることを痛感しました。
この辺りは、あまり最近のフロントエンド開発に慣れてない人にとっては、学習が難しい部分となりそうな予感がします。

この記事を書くにあたって、次の記事も参考にしています。

- [Working with Fonts with Webpack | Adrian Hall](https://shellmonger.com/2016/01/22/working-with-fonts-with-webpack/)
- [Using @font-face | CSS-Tricks](https://css-tricks.com/snippets/css/using-font-face/)
