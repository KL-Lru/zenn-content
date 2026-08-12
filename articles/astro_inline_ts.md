---
title: "Astro の inline script を無理矢理にでも TS で書く"
emoji: "🌟"
type: "tech"
topics: ["Astro", "TypeScript"]
published: true
---

## 始めに

自分用の学習ノートを記録として残すサイトでも立ち上げるか, と静的サイトを構成するために Astro を使い始めました.
その中でライトテーマ / ダークテーマに対応するに当たって, 初期ロード時のチラつきを防ぐべく, head に inline のテーマ適用のスクリプト仕込もうとした際のことです

```astro
<script is:inline>
import { applyTheme, getTheme, setTheme, useSystem } from '@/utils/theme.ts';

function observeSystemTheme() {
  const mq = window.matchMedia('(prefers-color-scheme: dark)');

  mq.addEventListener('change', () => {
    if (useSystem()) {
      applyTheme('system');
    }
  });
}

function init() {
  setTheme(getTheme());
  observeSystemTheme();
}

init();
</script>
```

このコードは動作しません. 

こういった inline script を TS で書いて動作させるための構成を無理やり組んだ備忘録です.

## inline script の制約

Astroの inline スクリプトはそのまま出力される HTML へと出力されます. このため, 次のような制約が課されています.

- import 文を処理することが出来ない
- TypeScript で記述することが出来ない

inline スクリプトは基本, ブラウザから処理できるように全てその場で JavaScript で記述する必要があります.

## 解決したかったもの

### FOUC

表示テーマ処理は, head 内などの位置で同期的に解析され, レンダリングが実行されるよりも前に実行される必要があります.
HTML 文書が完全に処理される前に, その処理をブロックしながら処理を実行することによって, いわゆる FOUC (Flash of Unstyled Content) の発生を防ぐためです.

Astro の通常の script 埋め込みは, 自動的に `type="module"` を付与します. この属性指定によってスクリプトが処理をブロックすることがなくなるため, 次のように記述した場合 FOUC を防ぐことができなくなります.

```astro:Main.astro
<script src="@/assets/scripts/init.ts"></script>
```

```html:output.html
<script type="module" src="/_astro/Main.astro_astro_type_script_index_0_lang.BVyFXi-c.js"></script>
```

### コード重複 / JS 記述

inline で記述 / 処理したい部分はテーマ処理の初期化部分のみです.
テーマ切り替えボタンを配置してテーマの切り替え処理を実行することも多くあり, こちらでも同じような処理を実行して指定されたテーマへの制御を行う必要があります.
この際 inline 側だけ import できないとなると, 初期化処理と切り替えボタン押下時で複数個所に同一のコードを配備しなければならなくなります.

また, このご時世に素の JavaScript を書きたいニーズはありません.

## 解決方法

解決する方法はとても単純明快です.
**埋め込みたいコードを import なしの JavaScript に変換して直接埋め込む** という方法を取ることが出来ます. ゴリ押しです.

### Plugin を作る

Astro は内部で  Vite を利用しています. そして Vite には拡張子などをもとに import する際の処理を追加できる Plugins 機構が存在します.

というわけで, 特定の suffix を付与した import に限り, その場で直接トランスパイルを実行した結果を import 時に返却する Plugins を記述し, 追加します. 今回は `?inline-bundle` がその suffix です.

```ts
import { type Plugin } from 'vite';
import { buildSync } from 'esbuild';

export function inlineTsPlugin(): Plugin {
  return {
    name: 'inline-bundle',
    load(id) {
      if (id.endsWith('?inline-bundle')) {
        const filePath = id.replace('?inline-bundle', '');

        const result = buildSync({
          entryPoints: [filePath],
          bundle: true,
          write: false,
          minify: true,
        });
        const bundledCode = result.outputFiles[0].text;

        return { code: `export default ${JSON.stringify(bundledCode)}`, map: null };
      }
    }
  };
}
```

TypeScript のエラーとならないよう, d.ts ファイルも追加しておきましょう.

```ts
declare module '*?inline-bundle' {
  const content: string;
  export default content;
}
```

### 実際に利用する

作成した plugins を astro.config にて利用するよう指定します.

```ts
export default defineConfig({
  vite: {
    resolve: {
      alias: {
        '@': '/src',
      },
    },
    plugins: [inlineTsPlugin()],
  },
  // ...
});
```

この import 処理を利用して, 実際に theme の初期化処理を埋め込むとこのようになります.

```astro
import initialize from "@/assets/scripts/init?inline-bundle";

---

<html lang="ja">
  <head>
    <meta charset="UTF-8" />
    <!-- ... -->

    <title>{title}</title>
    <script is:inline set:html={initialize} />
  </head>
  <body>
    <main class="container">
      <!-- ... -->
    </main>
  </body>
</html>
```

```ts:assets/scripts/init.ts
import { applyTheme, getTheme, setTheme, useSystem } from '@/utils/theme';

function observeSystemTheme() {
  const mq = window.matchMedia('(prefers-color-scheme: dark)');

  mq.addEventListener('change', () => {
    if (useSystem()) {
      applyTheme('system');
    }
  });
}

function init() {
  setTheme(getTheme());
  observeSystemTheme();
}

init();
```

:::details utils/theme.ts (参考, 埋め込んでいる対象処理)

```ts:utils/theme.ts
type Theme = 'light' | 'dark' | 'system';

export const DEFAULT_THEME: Theme = 'system';
export const THEME_KEY = 'theme';

export function useSystem() {
  return localStorage.getItem(THEME_KEY) === 'system' || !localStorage.getItem(THEME_KEY);
}

export function systemDark() {
  return window.matchMedia('(prefers-color-scheme: dark)').matches;
}

export function systemLight() {
  return window.matchMedia('(prefers-color-scheme: light)').matches;
}

export function systemTheme() {
  if (systemDark()) return 'dark';
  if (systemLight()) return 'light';

  // not reachable
  return DEFAULT_THEME;
}

export function getTheme() {
  const storedTheme = localStorage.getItem(THEME_KEY);

  if (storedTheme === 'light' || storedTheme === 'dark') {
    return storedTheme;
  }

  return 'system';
}

export function setTheme(theme: Theme) {
  localStorage.setItem(THEME_KEY, theme);

  applyTheme(theme);
}

export function applyTheme(theme: Theme) {
  const html = document.documentElement;
  const currentTheme = theme === 'system' ? systemTheme() : theme;

  html.classList.remove('light', 'dark');
  html.classList.add(currentTheme);
}
```

:::

これで bundle 済みの JavaScript 記述が script タグ内に展開され, 初期化処理が埋め込まれます.

```html
<script>
"use strict";(()=>{var i="system",t="theme";function o(){return localStorage.getItem(t)==="system"||!localStorage.getItem(t)}function a(){return window.matchMedia("(prefers-color-scheme: dark)").matches}function h(){return window.matchMedia("(prefers-color-scheme: light)").matches}function l(){return a()?"dark":h()?"light":i}function m(){let e=localStorage.getItem(t);return e==="light"||e==="dark"?e:"system"}function n(e){localStorage.setItem(t,e),r(e)}function r(e){let s=document.documentElement,c=e==="system"?l():e;s.classList.remove("light","dark"),s.classList.add(c)}function u(){window.matchMedia("(prefers-color-scheme: dark)").addEventListener("change",()=>{o()&&r("system")})}function d(){n(m()),u()}d();})();
</script>
```

## メリット

- HTML に直接埋め込む inline コードを TypeScript ファイルとして記述することができる
  - 単純に保守性の向上
- 必要に応じて minify などの制御を追加できる
  - 単純な inline js より軽量化できる

## デメリット

- esbuild だけで単純に処理できないようなものはそのビルド工程を制御する必要が出てくる
  - inline で複雑怪奇な処理を記述する必要があまりないので, そもそも困るケースも少ない
- ビルド対象が大量に増えると個別ビルドの分重たくなる
  - そも inline はその場で実行しなければならない理由があるもの以外するべきではない

## まとめ

思ったよりサクッとできた一方で実用性はあるのか...? と言われるとまた微妙なとこだなーという感覚がちょっとあります.
FOUC 対策は Firefox が [blocking="render"](https://developer.mozilla.org/ja/docs/Web/API/HTMLScriptElement/blocking) に対応したらこっちにしましょう.
