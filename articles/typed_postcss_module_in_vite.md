---
title: "Vite + Open Props + CSS Modules + TypeScriptでゴリゴリにCSSを書き殴る"
emoji: "🎨"
type: "tech"
topics: ["Vite", "TypeScript", "PostCSS", "CSSmodules", "React"]
published: false
---

## 概要

React で CSS をゴリゴリ記述したく, Open Props + CSS Modules の構成を組んだ備忘録です.

Open Props は CSS の多様な変数やカスタムメディアクエリ等を提供してくれるライブラリであり, デザインの統一性を保ったり, 記述の煩雑さを低減してくれます.
CSS Modules は CSS をスコープ化し, CSS Class 名の衝突などを防いでくれます.
これらを利用していくことで, 効率的に Component 向けの CSS を書いていくことができます.

しかし, CSS Modules はそれ単体では型情報がなかったり, Open Props はカスタムメディアクエリの利用にプラグインの導入が必要だったりと迷いポイントがじんわりあります.
TypeScript や PostCSS のプラグインを利用して, これらを解消しつつ, 快適に CSS を書いて Component を構築していけるところを目指します.

## この記事執筆時点で利用したライブラリ

| library              | version |
| -------------------- | ------- |
| Vite                 | ^5.2.0  |
| React                | ^18.2.0 |
| TypeScript           | ^5.2.2  |
| Typed CSS Modules    | ^0.9.1  |
| PostCSS              | ^8.4.38 |
| PostCSS Custom Media | ^10.0.6 |
| PostCSS Global Data  | ^2.1.1  |
| Open Props           | ^1.7.4  |

## Open Props の導入

https://open-props.style/

### CSS Import

CSS 内にて次のような記述を入れることですべての変数定義が読み込まれます.

```css
@import "open-props/style";
```

:::message
`open-props/style`を import した場合, import した CSS ファイルに変数定義が展開されるため, CSS Modules として扱う CSS 等で記述してしまうと, プロジェクト全体で変数が重複定義される場合があります.
基本的にはグローバルな CSS ファイルでのみ import し, それ以外の CSS ファイルでは import しないようにすることをおすすめします.
:::

このパスについては open-props の `package.json` の `exports` にどの CSS が参照されているか記述されています. 一覧がほしい場合や, Import されている対象が知りたい場合はそちらを参照しましょう.

### Custom Media 対応

Open Props はカスタムメディアクエリも提供してくれます.
PostCSS でカスタムメディアクエリを利用する場合は, プラグイン`postcss-custom-media`を追加する必要があります.

```ts:postcss.config.ts
import postcssCustomMedia from 'postcss-custom-media';

export default {
  plugins: [
    postcssCustomMedia(),
  ],
};
```

```ts:vite.config.ts
import { defineConfig } from 'vite';

import react from '@vitejs/plugin-react';
import postcss from './postcss.config';

export default defineConfig({
  plugins: [react()],
  css: {
    postcss
  },
});
```

:::message
`postcss.config.ts`は Vite に自動読み込みされますが, CommonJS 形式を強制されたり, ES module 形式で記述するために拡張子を`mts`にしたりと迷走ポイントが微妙に存在するので, 直接 ts ファイルを import してしまう形をおすすめします.
:::

#### CSS Import

変数定義の読み込みと同様に, カスタムメディアクエリも import を記述することで利用できるようになります.

```css
@import "open-props/media";
```

この形式を取る場合注意したいのは, **カスタムメディアクエリはこの`@import`は記述した CSS ファイルのみで利用できる**という点です.
これは PostCSS がそれぞれの個々の CSS ファイルについて処理を行うようになっていることに起因するものです. この`@import`が記述された CSS ファイル中ではカスタムメディアクエリは実際のメディアクエリに置き換えられますが, 他のファイルはそのままの状態で残り, ブラウザに解釈されることはありません.

#### PostCSS Global Data

各 CSS での Import を毎回記述しなくてはならないのは手間なので, 別の手段でカスタムメディアクエリを処理します.
`@import`を記述する代わりに PostCSS のプラグイン`postcss-global-data`を追加することで, このカスタムメディアクエリをグローバルに処理することができるようになります.

```ts:postcss.config.ts
import { createRequire } from 'node:module';
const require = createRequire(import.meta.url);

import postcssGlobalData from '@csstools/postcss-global-data';
import postcssCustomMedia from 'postcss-custom-media';

export default {
  plugins: [
    postcssGlobalData({ files: [require.resolve('open-props/media')] }),
    postcssCustomMedia(),
  ],
};
```

このような設定を記述する場合は, Vite にて [`require.resolve`が直接利用できない](https://ckeditor.com/docs/ckeditor5/latest/getting-started/legacy/advanced/alternative-setups/integrating-from-source-vite.html#vite-configuration)点に注意が必要です.

### VS Code の拡張

- [CSS Var Complete](https://marketplace.visualstudio.com/items?itemName=phoenisx.cssvar) を入れておく
- settings.json に記述を追加しておく

の 2 ステップで完了します.

```json
  "cssvar.files": [
    "./node_modules/open-props/open-props.min.css",
  ],
  "cssvar.ignore": [],
  "cssvar.extensions": [
    "css", "postcss", "jsx", "tsx"
  ]
```

## CSS Modules の導入

Vite では CSS Modules はデフォルトでサポートされています.
コンポーネントから参照する CSS ファイルはすべて`module.css`で終わるようにしておきましょう.

## Typed CSS Modules の導入

Vite で CSS Modules を単に導入しただけでは, 型情報が `CSSModuleClasses` となります.
内容は`{ readonly [key: string]: string }`であり, どのような CSS Class が存在するか不明瞭なままであり, 存在確認等は実施されません.

Typed CSS Modules を利用することで, CSS Modules から生成される`.d.ts`ファイルに型情報が追加することができます.

:::message
めちゃくちゃ似ているライブラリに`typed-scss-modules`がありますが, Open Props とかの過程で PostCSS を入れており, 追加で同様の役割を担う SCSS を入れるのは避けたい意図で当記事では`typed-css-modules`を利用しています.
:::

以下実コードは`src`ディレクトリ, `generated`ディレクトリに型定義ファイルを生成するものとなっています.

### 実行時に型を生成する

#### `npm-run-all`を利用する場合

https://www.npmjs.com/package/npm-run-all

こちらを利用することで複数の npm script を一括実行することができます.
`typed-css-modules`では`tcm`コマンドで型定義ファイルを生成できるので, これを利用して生成できます.

```json:package.json
{
  // ...
  "scripts": {
    "dev": "run-p dev:*",
    "dev:vite": "vite",
    "dev:css": "tcm src --watch --outDir generated",
   // ...
  }
  // ...
}
```

#### Vite Plugin にしてしまう場合

Vite Plugin にしてしまうことで, 出力時のメッセージを調整したり, 初期ロード時に不要になった型定義ファイルを削除したり等の処理を自由に追加することができます.
独自カスタムをかけたい場合はこちら.

:::details 実装例

```ts:tcm.plugin.ts
import type { Plugin } from 'vite';
import { DtsCreator } from 'typed-css-modules/lib/dts-creator';
import { run } from 'typed-css-modules/lib/run';
import path from 'node:path';

class TcmTranspiler {
  private creator: DtsCreator;
  private CSS_MODULE_REG = /\.module\.css$/;

  constructor() {
    this.creator = new DtsCreator({
      searchDir: './src',
      outDir: './generated',
    });
  }

  is_css_module(path: string) {
    return this.CSS_MODULE_REG.test(path);
  }

  async processAll() {
    await run('./src', {
      outDir: './generated',
      silent: true,
    });
  }

  async create(source_path: string) {
    try {
      const content = await this.creator.create(source_path, undefined, true);
      await content.writeFile();

      console.log(`[TCM] Generate ${content.relativeOutputFilePath}`);
    } catch (e) {
      console.error(`[Error] ${e}`);
    }
  }

  async delete(source_path: string) {
    try {
      const content = await this.creator.create(source_path, undefined, true, true);
      await content.deleteFile();
      console.log(`[TCM] Delete ${content.relativeOutputFilePath}`);
    } catch (e) {
      console.error(`[Error] ${e}`);
    }
  }
}

function tcmPlugin(): Plugin {
  const transpiler = new TcmTranspiler();

  return {
    name: 'typed-css-modules',

    async buildStart() {
      await transpiler.processAll();
    },

    async watchChange(id, changeType) {
      if (!transpiler.is_css_module(id)) {
        return;
      }

      const event = changeType.event;
      if (event === 'create' || event === 'update') {
        await transpiler.create(id);
      } else if (event === 'delete') {
        await transpiler.delete(id);
      }
    },
  };
}

export default tcmPlugin;
```

:::

### Type を適切に読み込ませる

`typed-css-modules` はデフォルトで css と同一ディレクトリに型定義ファイルを生成します.
これがそこそこ邪魔なので前述のように `generated` ディレクトリに生成するようにしていますが, これでは型情報が読み込まれません.

TypeScript には幸いにも「複数の異なるディレクトリを同一ディレクトリとみなして処理する」というオプションがあります. 皆様ご存知の`rootDirs`です.
生成ファイルとソースファイルを別ディレクトリにわけ, それをそれぞれ`rootDirs`に指定することで, 生成ファイルがソースファイルと同ディレクトリ内にあるかのように型を参照できるようになります.

```json:tsconfig.json
{
  "compilerOptions": {
    // ...
    "rootDirs": ["src", "generated"]
    // ...
  }
}
```

## 出来上がった環境

サンプルとして, アイコン付きボタンのコンポーネントを記述してみます.

### サンプルソース

```css:index.module.css
.button {
  display: inline-flex;
  flex-wrap: nowrap;
  padding: var(--size-2);
  border: var(--border-size-1) solid color-mix(in srgb, currentColor 40%, transparent) ;

  align-items: center;
  justify-content: center;
  gap: var(--size-1);
  border-radius: var(--size-1);

  @media (--OSlight) {
    color: var(--gray-9);
    background-color: var(--gray-1);

    &:hover {
      background-color: var(--gray-3);
    }
  }

  @media (--OSdark) {
    color: var(--gray-1);
    background-color: var(--gray-9);

    &:hover {
      background-color: var(--gray-8);
    }
  }

  transition: background-color .3s;
}

.button__icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;

  border-radius: 50%;
  width: var(--font-size-1);
  height: var(--font-size-1);
}

.button__label {
  display: inline-block;
  font-size: var(--font-size-0);
}
```

```tsx:iconButton.tsx
import { MouseEventHandler, ReactNode } from 'react';
import styles from './index.module.css';

type Props = {
  icon: ReactNode;
  label: ReactNode;
  onClick?: MouseEventHandler<HTMLButtonElement>;
};

export function IconButton({ icon, label, onClick }: Props) {
  return (
    <button className={styles.button} onClick={onClick}>
      <span className={styles.button__icon}>{icon}</span>
      <span className={styles.button__label}>{label}</span>
    </button>
  );
}
```

### レンダリングされる結果

この Component に devicon などを指定してやると次のような rendering が得られます.

![](/images/typed_postcss_module_in_vite/button_light.png)
_Light Mode_
![](/images/typed_postcss_module_in_vite/button_dark.png)
_Dark Mode_

### ビルド後の CSS

コンフリクトしそうな CSS Class を記述していますが, CSS Module を使っていることで, 他の Component と衝突することはないので気にする必要はありません.
ビルドすると次のような CSS が生成され, それぞれの Component に適用されます.

```css:after_build.css
._button_1krtr_1{align-items:center;border:var(--border-size-1) solid color-mix(in srgb,currentColor 40%,transparent);border-radius:var(--size-1);display:inline-flex;flex-wrap:nowrap;gap:var(--size-1);justify-content:center;padding:var(--size-2);transition:background-color .3s}
@media (prefers-color-scheme:light){._button_1krtr_1{background-color:var(--gray-1);color:var(--gray-9)}._button_1krtr_1:hover{background-color:var(--gray-3)}}
@media (prefers-color-scheme:dark){._button_1krtr_1{background-color:var(--gray-9);color:var(--gray-1)}
._button_1krtr_1:hover{background-color:var(--gray-8)}}
._button__icon_1krtr_33{align-items:center;border-radius:50%;display:inline-flex;height:var(--font-size-1);justify-content:center;width:var(--font-size-1)}
._button__label_1krtr_43{display:inline-block;font-size:var(--font-size-0)}
```

### CSS 変数の補完

CSS を記述した時点で, 各種 CSS 変数の値の内容は参照することができるようになっています.
色等についても, エディタ補完で色情報が表示されるようになっているのが確認できます.

![](/images/typed_postcss_module_in_vite/css_variables.png)

サイズを指定したい場合も, `var(--size`あたりまで入力したところでサジェスト機能を利用できます. 困ったらここから選べば良い.

![](/images/typed_postcss_module_in_vite/value_suggest.png)

### CSS Modules の型情報

また, CSS Modules を参照した先の CSS Class 名は TypeScript から認識され, 確認可能な状態になっています.

![](/images/typed_postcss_module_in_vite/css_class_type.png)

この状態で CSS Class 名を変更するなどすると, TypeScript からエラーが発生するようになり, Typo しても一目瞭然になります.
これでもう誤った CSS Class 名を使うことはないでしょう.

```diff
- .button__icon {
+.button__icon_level2 {
  display: inline-flex;
  align-items: center;
  justify-content: center;

  border-radius: 50%;
  width: var(--font-size-1);
  height: var(--font-size-1);
}
```

![](/images/typed_postcss_module_in_vite/css_class_type_error.png)

## まとめ

Vite で Open Props + CSS Modules + TypeScript を利用して, いい感じに CSS をゴリゴリに記述する方法を紹介しました.

グローバルに CSS Class をぶちまけることなく, Open Props で必要な変数の共通化は行い, Component 毎にスコープ化された CSS を記述し, TypeScript で CSS Class の型情報を補完しつつ, Component をガリガリ構築していく.
個人的には CSS-in-JS や, Tailwind CSS などよりもこっちのほうが好みです.
