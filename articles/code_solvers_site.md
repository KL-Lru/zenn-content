---
title: "Rust で書いた競プロのコードを解説付きで rustdoc で公開する"
emoji: "📖"
type: "tech"
topics: ["Rust", "競プロ"]
published: true
---

## 何があったのか

競技プログラミング, 縮めて競プロ.
与えられた問題を解くために, プログラムを書く競技です.

以前から「競プロのコード解説とかどこかでやりたい！」という気持ちがあり, はてなブログなどで記事をすごく雑に書いていた時期もありました. ですがなかなか続かないのが現実でした.

- GitHub に投げたソースコードとその解説のブログを同期するのが単純に手間
- ブログサイトに投稿するまでの道のりが長い

## 実現したいこと

「競プロをやった後, メモとコードを GitHub に投げたらほぼコストなしで解説公開サイトが錬成される」が理想です.

なので次のような点を重視して考えます.

- 競プロで利用するソースコードの構成には影響を与えない
- 解説は Git 管理が容易な Markdown で記述する
- ソースコードと解説は GitHub 上で管理する
- GitHub 上に push されたソースコードと解説の組を, 自動的にサイトに反映する
- 閲覧しやすさを考慮してソースコードと解説は同一のページに展開する

## できたもの

rustdoc にてこんな感じに展開されます.

https://kl-lru.github.io/atcoder-rs/abc_404/d/index.html

元ファイルは次の 2 つです.

@[card](https://github.com/KL-Lru/atcoder-rs/blob/main/abc_404/src/bin/d.rs)
@[card](https://github.com/KL-Lru/atcoder-rs/blob/main/abc_404/docs/d.md)

## 実現への道のり

### 1. ソースコード構成の整理

Rust で競プロをやる場合, 多くのソースコードは次のような構成になります.

```
workspace/
 ├── Cargo.toml
 ├── Cargo.lock
 └── (コンテスト名)/
     ├── src/
     │   └── bin/
     │      ├── a.rs
     │      ├── b.rs
     │      ├── c.rs
     │      └── d.rs
     └── Cargo.toml
```

この構成を取ると, ライブラリなどの version は workspace 単位で固定化でき, コンテストが個別の crate として管理できます. `target` ディレクトリが Workspace 単位で管理されるため, コンテストごとに `target` ディレクトリが作成されることもなくなります.

よく`src/main.rs`に配置する構成例を見ますが個人的には反対です. 途中まで書いたけど詰まったから次の問題! としたいときに`src/main.rs`だけしかない状態だと余計な手間が増えます.

`src/bin`以下にファイルを格納することで `bin` ターゲットとして認識され, `cargo run --bin a` のように `a.rs` を実行できます.

### 問題 1: rustdoc が実行できない問題

Rust のコードの公開や解説の掲示, というところでまずはじめに考えるのが, `rustdoc`を使ってドキュメントを生成することです. `rustdoc`は ドキュメント生成ツールであり, ソースコードに記述されたドキュメントコメントを元に, ドキュメントを生成してくれます.

また, `include_str!`といったマクロを利用することで, 外部の Markdown ファイルをドキュメントとして取り込むこともできます. この機能を利用すれば, ソースコードはほぼ提出したものと同一を保ちながら, 解説を Markdown で記述できるでしょう.

さて, `rustdoc`を実行してみましょう!

```
error: document output filename collision
The bin `a` in package `abc_358 v1.0.0 (/home/lru/work/atcoder-rs/abc_358)` has the same name as the bin `a` in package `abc_001 v1.0.0 (/home/lru/work/atcoder-rs/abc_001)`.
Only one may be documented at once since they output to the same path.
Consider documenting only one, renaming one, or marking one with `doc = false` in Cargo.toml.
```

滅びました.
`rustdoc`は, 同一の名称の`bin`ターゲットが複数存在する場合, どれか 1 つしかドキュメントを生成できません.

前述の構成では各コンテストの crate 毎に`a`, `b`といった同一名称の`bin`ターゲットが存在するため, `rustdoc`はどれか 1 つしかドキュメントを生成できません.
ChatGPT くんに助けを求めると, 「1 つ 1 つの crate でドキュメント生成してくっつければいいじゃん!」とか言われます. そんなことしたくないので無視することにします.

競プロのやりやすさに影響が出るので, 構成の変更は可能な限りしたくありません...
ということで黒魔術じみたオートビルドへの旅が始まります.

### 2. `src/bin`以下のファイルをモジュール化する

`rustdoc`でドキュメントが生成できなかったのは, 同一の名前の`bin`ターゲットが複数存在したからです. このエラーとなる挙動は, `rustdoc`で`bin`ターゲットも含めてドキュメントを構築した場合に限られます.

要するに「ライブラリ」として, 「コンテストの crate 内にあるモジュール」として扱ってやれば, ドキュメント生成が可能です.

なので次のようなファイルを配置してやれば, `rustdoc`は無事に動きます.

```rust:src/lib.rs
pub mod bin {
    pub mod a;
    pub mod b;
    pub mod c;
    pub mod d;
    // ...
}
```

```bash
$ cargo doc --lib
```

### 問題 2: 手動調整が必要な`lib.rs`

前述の`src/lib.rs`では, 既に作成済みの`a.rs`などのファイルを指定し, モジュール化しています.
しかし GitHub 上に上げる見込みのコードは**解けた問題についてのコードのみ**です.

記述するモジュールは, はじめから全問分を固定で入れておけば良いわけではなく, 解法を記述した後に手動で追加する必要があります. これは手間で面倒です.

この手間を減らすために, `src/lib.rs`を自動生成することにしました.
`build.rs`の節にて後述します.

### 3. ソースコードに解説を埋め込む

`rustdoc`では, `include_str!`マクロを利用することで, 外部のファイルをドキュメントとして取り込むことができます.

```rust
#![cfg_attr(doc, doc = include_str!("../docs/a.md"))]

fn main() {
    // ...
}
```

このように記述することで, `a.rs`のドキュメントとして, `docs/a.md`を取り込むことができます.

`cfg_attr`は`rustdoc`を実行した際のみ有効になる属性を付与します.
これで実行時には寄与しないため, 最悪このまま提出しても問題ありません.

### 問題 3: `rustdoc`の実行時に解説ファイルが存在しない場合

A 問題など, そもそも解説いる...? みたいな問題もあり, 解説が不要なことも多いでしょう.
その場合 解説ファイルは存在しないことになりますが, `rustdoc`は存在しないファイルを指定するとエラーになります.

```bash
error: couldn't read `/home/lru/work/atcoder-rs/abc_358/docs/a.md`: No such file or directory (os error 2)
```

手動で解説ファイルの埋め込みを行うのは避けたいですが, 固定で記述するとエラーとなってしまう可能性が生じるため, これも自動化したくなります.

### 問題 4: ソースコード自身の埋め込み時に余計なものがつく

解説と実コードは同一のページで閲覧できるようにしたいので, markdown 以外にソースコード自体も埋め込む必要があります.
しかし, 自身を埋め込もうとすると, このドキュメントを錬成するために記述した, 各種無駄コードも共に展開されます.

````rust
//! ↓ こいつらも展開されてしまう
#![cfg_attr(doc, doc = include_str!("../docs/a.md"))]
#![cfg_attr(doc, doc = "```rust")]
#![cfg_attr(doc, doc = include_str!("./a.rs"))]
#![cfg_attr(doc, doc = "```")]

fn main() {
    // ...
}
````

提出したコードは可能な限り変更せず載せたい意図もあり, これは不適です.
除去された状態のファイル内容を埋め込む必要があります.

### 4. `build.rs`の爆誕

ここまでで見えた問題を整理すると, 次のようになります.

- `bin`ターゲットではなく, `lib`ターゲットとしてドキュメントを生成したい
- `lib.rs`は自動生成したい
- 解説ファイルが存在しない場合は, 埋め込みをスキップしたい
- ソースコード自身を埋め込む際には, そのコード本体のみを埋め込みたい

これらの問題を解決するために, `build.rs`を作成します.
先に全体像だけ示すと, 次のような流れになります.

![](/images/code_solvers_site/output.png =400x)
*build.rs にやらせること*

詳細なコードはリポジトリから参照ください.

https://github.com/KL-Lru/atcoder-rs/blob/main/build.rs

https://github.com/KL-Lru/atcoder-rs/blob/main/lib/docgen/src/generator.rs

#### 4-1. アウトプットの制御

`build.rs`は, Cargo のビルド時に実行されるビルドスクリプトです. このビルドを実行する際, **ビルド時にしか使わないファイル**を作成 / 配置できます.

ビルド時使用ファイルについては, 環境変数`OUT_DIR`の示すディレクトリ配下に錬成します. そして, その生成したファイルを, 実ソースコード内に展開できます.

```rust
include!(concat!(env!("OUT_DIR"), "展開対象ファイル.rs"));
```

今回の満たしたい要求のため, 次のように出力を制御します.

- `lib.rs`は`src`配下に出力する
  - `src/lib.rs`がない状態だと, `cargo doc`で`lib`ターゲットと認識されないため
  - 読み込み対象とするモジュールは`OUT_DIR`配下で制御する
- 解説の埋め込みやソースコードの埋め込みは, `OUT_DIR`配下で制御する
  - 大本のソースコードを変更したくないため
  - ドキュメントに展開するソースコードにだけ, 埋め込み処理を行う

#### 4-2. `lib.rs`の自動生成

`lib.rs`は `cargo doc`を実行した際に, `lib`ターゲットとして認識されるために必要なため, `src`配下へ直接生成します.
この際, 宣言するモジュールは動的に変更する必要があるため, `build.rs`で読み込む対象のファイルを錬成します.

```rust:src/lib.rs
#![doc(html_no_source)]
include!(concat!(env!("OUT_DIR"), "/modules.rs"));
```

```rust:target/debug/build/xxxxx/out/modules.rs
// 存在しているbinターゲットのみ列挙する
pub mod a;
pub mod b;
pub mod c;
pub mod d;
pub mod e;
pub mod f;
pub mod g;
```

ソースコードへのリンクを提供してしまうと, `build`時に使ったファイルがソースコードとして扱われてしまうことがあるため, `html_no_source`を指定しています. どのみち提示したいソースコードはドキュメント上に展開するので特に問題は発生しません.

#### 4-3. 解説埋め込みソースコードの自動生成

記述したソースコードはそのままに, ドキュメントの埋め込み記述だけを記述したコードを生成します.

埋め込み記述は動的に生成するため, ドキュメントファイルの有無で分岐できます.
また, 埋め込むソースコードとして, 埋め込み前のソースコードを指定することで, 余計な記述を付加せず埋め込めます.

```rust:src/bin/a.rs
fn main() {
    // ...
}
```

````rust:target/debug/build/xxxxx/out/a.rs
// mdは解説ファイルがある場合のみ
#![cfg_attr(doc, doc = include_str!("/absolute path/to/docs/a.md"))]
#![cfg_attr(doc, doc = "```rust")]
#![cfg_attr(doc, doc = include_str!("/absolute path/to/src/bin/a.rs"))]
#![cfg_attr(doc, doc = "```")]
````

前述の`modules.rs`に記述されている`pub mod a;`は, このファイルを参照して取り込みます.
そのため, `lib`ターゲットとしてビルドした際には, こちらの`OUT_DIR`配下のファイルが参照されるようになります.

### 5. GitHub Actions で自動デプロイ

いつものです.

workspace の index page がほしかったので `nightly`を利用します.
また, `cargo doc`を実行する前に, crate のビルドを行う必要があるため, `cargo build`をはさみます.

https://github.com/KL-Lru/atcoder-rs/blob/main/.github/workflows/rust.yml

## おしまい

これで競プロをやった後, Push するだけでコードが公開されるようになりました.
やる気が出たらまた競プロやりましょうかね.
