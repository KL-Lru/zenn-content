---
title: "Not `make` do it, `just` do it!"
emoji: "🚀"
type: "tech"
topics: ["Makefile", "Justfile"]
published: true
---

# TL;DR

- `Makefile`は「ビルドするタスク」を記述するためのもの
- ビルドしないタスクも記述できるが, `make`だとちょっと残念な点がある
- ただのタスクを記述するだけなら`make`よりも`just`を使ってみるといいかもしれない

# `make` It!

`make`はプログラムやソフトウェアをビルドするコマンド/ツールです. [^1]
`Makefile`に記述された, ターゲットに関するレシピを実行することで, プログラムやソフトウェアのビルドやテスト, デプロイなどを行います.
これを利用することで, 実行しなければならないコマンド群を記録しておき, 依存関係に従って 1 コマンドで必要な処理を実行できます.

```makefile
hello: hello.cpp
	g++ -o hello hello.cpp

hello.cpp:
	echo '#include<iostream>' >> hello.cpp
	echo 'using namespace std;' >> hello.cpp
	echo 'int main() { cout << "Hello World!" << endl; return 0;}' >> hello.cpp

.PHONY: clean
clean:
	rm -f hello hello.cpp
```

上記記述の場合, `hello`ターゲットは`hello.cpp`に依存しており, `hello.cpp`のビルドを先に実行しなくてはならないことを示しています.

```bash
make hello
# echo '#include<iostream>' >> hello.cpp
# echo 'using namespace std;' >> hello.cpp
# echo 'int main() { cout << "Hello World!" << endl; return 0;}' >> hello.cpp
# g++ -o hello hello.cpp
```

無事に`hello`ターゲットのビルドが完了し, 実行可能バイナリの生成が行われました.

```bash
./hello
# Hello World!
```

不要になったバイナリや生成されたソースコードを削除する場合は, `clean`ターゲットを実行します.

```bash
make clean
# rm -f hello hello.cpp
```

今回注目したいのは最後の「何も生成せず, 記述されたタスクのみを実行した」という仮想(PHONY)ターゲット`clean`です.

# `make` Do It...?

`Makefile`には「実際に何もビルドしないターゲット」を定義できます.
これに着目し, 「コマンドがクソ長くて覚えるのもダルいし一発でできたほうがカッコいいしもう`Makefile`で書いちゃおうぜ」としている派閥があります. (諸説あり)

良い例が`docker compose`です.

```yaml
services:
  app:
    image: nginx:latest
# その他有象無象のコンテナたち...
```

```makefile
up:
	docker compose up -d

down:
	docker compose down

console:
	docker compose exec app bash

.PHONY: up down console
```

これにより劇的にコマンドが短縮されました.

```bash
make up
# docker compose up -d

make down
# docker compose down

make console
# docker compose exec app bash
```

上記例は単発コマンドのみですが, 複数のコンテナにまたがった初期化処理を行わなければならないといった場合には短縮量はかなりのものになります.

```makefile
init:
	docker compose exec app bundle install
	docker compose exec app yarn install
	docker compose exec app bundle exec rails db:create db:migrate db:seed
```

```bash
make init
```

## 残念ポイント

この方法で利用する場合, `Makefile`は便利なツールですが, いくつか残念な点があります.

### PHONY

`Makefile`は「ビルドするもの」を記述するためのものです.
何もビルドしないタスク実行用途では仮想ターゲットであることを明示するため`.PHONY`を記述する必要があります.

```makefile
.PHONY: up down console
```

この指定を忘れてしまった場合, 該当するターゲットと同一の名称のファイルまたはディレクトリが存在すると, 該当のタスクが実行できなくなります.
これはターゲットに依存しているファイルが存在せず, 更新すべきタイミングを`make`が識別できないことに起因するものです.
依存関係がないものは, 一度作成してしまえばそれ以後は再作成する必要がないものとみなされます.

```bash
touch console

make console
# make: 'console' is up to date.
```

### Arguments

`Makefile`では, コマンドライン引数を利用するためには, 変数として値を格納して実行する必要があります.
例えば前述の`make up`の例で, 特定のコンテナのみ起動したい場合には, 以下のように記述する必要があります.

```makefile
up:
	docker compose up -d $(CONTAINER)
```

```bash
make up CONTAINER=app
```

うっかり `docker compose up -d app`を打つんだ!!と変数に格納するのを忘れると, 別のターゲットが実行されることになります.

```bash
make up app
# make: *** No rule to make target 'app'.  Stop.
```

渡したい引数が 1 つならこれで問題ありませんが, 複数になってくると記述量は増え, 当初「コマンド入力を短くする」という目標に相反する状態となっていきます.

### Indent

`Makefile`では, ターゲットに対するレシピの記述は Tab でインデントしなければなりません.

```makefile
# NG
hello: hello.cpp
        g++ -o hello hello.cpp

# OK
hello: hello.cpp
	g++ -o hello hello.cpp
```

見た目上かなり分かりづらく, 初学者が「なんか動かなくなったんですけど...」となる 1 つの要因です.
近年のインデント事情を鑑みると.

|言語|インデント|
|:--|:--|
|Rust|The Rust Style Guide [^2] にてSpaceのみ利用指定. rustfmtにてSpaceの利用を強制|
|C++|LLVM[^3]やGoogle[^4]のコーディングスタイルではSpaceのみを利用指定|
|Python|PEP8[^5]でSpaceを強く推奨|
|Ruby|Ruby Style Guide[^6]にてSpaceのみを利用指定|

といったように, Tab ではなく Space にすることが圧倒的に多くなっています.
エディタの設定を特に行わないまま他のプログラムと同様に`Makefile`を書くと, 思わぬ罠に引っかかります.
(今この記事を記述している私も, 今 Makefile のコードブロックを書くときだけインデントモードを Tab にしています)

# `just` Do It!

そこで**コマンドランナー** `just`の登場です.

https://github.com/casey/just

`Makefile`のような定義ファイル`Justfile`を作成し, 記述されたレシピに対応する処理を実行できます.

```makefile
up:
    docker compose up -d

down:
    docker compose down

console:
    docker compose exec app bash
```

```bash
just up
# docker compose up -d

just down
# docker compose down

just console
# docker compose exec app bash
```

## `make` vs `just`

### ビルド vs タスク

`just`はビルドツールではないため, `.PHONY`の指定は不要です.
レシピ名と同一のファイルやディレクトリがあったとしても, それを無視してレシピに記述された処理を実行します.

```bash
touch console

just console
# docker compose exec app bash
```

依存関係指定はできますが, 依存するレシピの名称を記述する形になります.
特定のファイルの有無による制御はできません.

```makefile
# レシピ名に"."は含められないので, hello.cppとは記述できない
hello: hello_cpp
        g++ -o hello hello.cpp

hello_cpp:
        echo '#include<iostream>' >> hello.cpp
        echo 'using namespace std;' >> hello.cpp
        echo 'int main() { cout << "Hello World!" << endl; return 0;}' >> hello.cpp

clean:
        rm -f hello hello.cpp
```

これは`make`の利点の 1 つである「再ビルドしなくても良い物はビルドしない」という制御が欠落することを意味します.

```bash
# hello.cpp, hello の binary が作成される
just hello

# 既に存在しているhello.cppに追記, hello の binaryを生成しようとして重複したmain関数でコンパイルエラー
just hello
```

```bash
# (記事冒頭のMakefile前提)
# hello.cpp, hello の binary が作成される
make hello

# 依存している対象が更新されていない場合, 何もしない
make hello
```

タスクの実行に際しては`just`は余計な指定が不要になりますが, ビルドを行って成果物が生じる場合は`make`の方が良いでしょう.

### Arguments

`just`はコマンドライン引数をそのまま利用できます.

```justfile
up *args:
    docker compose up -d {{ args }}
```

```bash
just up app
# docker compose up -d app
```

他にもデフォルト値を設定したり.

```justfile
up *args="app":
    docker compose up -d {{ args }}
```

```bash
just up
# docker compose up -d app
```

位置引数として展開して複数のコマンドで使ったりもできます.

```justfile
set positional-arguments

sample a b c:
    echo "a: $1"
    echo "b: $2"
    echo "c: $3"
```
```bash
just sample 3 2 1
# echo "a: $1"
# a: 3
# echo "b: $2"
# b: 2
# echo "c: $3"
# c: 1
```

この点に関しては`just`に軍配が上がります.

### Indent

`just`は Space でも Tab でもインデントを認識します.

```justfile
# OK
up:
        docker compose up -d
# OK
up:
	docker compose up -d
```

ややこしい部分が減るのはそれだけで嬉しいものです.

### その他

他にも`just`にはタスク実行に際して便利な機能がいくつかあります.

- 各タスクのドキュメント記述とヘルプ表示
    - `just --list`にて, どんなレシピがあるのか, 何をするものなのかを表示できます
- 実行シェル/言語の指定
    - Shell 以外に Python 等も指定可能
- `.env`からの環境変数読み取り
    - `just`打つ前に export しなくちゃ...の手間なし
- サブディレクトリからも実行可能
    - `just`を実行したディレクトリから親ディレクトリを探索
- 変数定義と展開

詳しくはリポジトリのドキュメントを参照ください.

## 雑感

次のような用途/PJ の場合, 利用検討の余地はありそうです.

- 個人の手元環境の整備に利用する
    - HOME においておけばサブディレクトリから打てるので汎用性が高い
    - alias や shell 関数を書くより手間が少なく済む
- ビルド対象が存在しない / `make`で依存関係管理をそもそも期待していない
- タスク一覧表示や dotenv 等の機能が欲しい
- タブ文字に破壊衝動を覚えたことがある

これらに当てはまらない場合には素直に`make`を使うほうがおそらく良いです.
また既に`Makefile`を利用している状態での導入はやめておくべきです.
絶対どっちのコマンドだっけ...となるので.

# まとめ

生成物を伴わないタスクであれば`just`を利用すると, タスクを簡素にかつ便利に記述できます.
逆にビルド処理が必要となるタスクの場合は, 依存関係解決の関係から`make`のほうがまだ利点は多いでしょう.

あなたが書きたかったのは「生成物のビルド処理を実行する」`make`か, 「ただタスクを実行する」`just`か, どちらだったでしょうか?

Not `make` do it, `just` do it!

[^1]: [GNU make](https://www.gnu.org/software/make/manual/make.html)
[^2]: [The Rust Style Guide](https://doc.rust-lang.org/nightly/style-guide/index.html?highlight=tab#indentation-and-line-width)
[^3]: [LLVM Coding Standards](https://llvm.org/docs/CodingStandards.html#whitespace)
[^4]: [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html#Spaces_vs._Tabs)
[^5]: [PEP 8 -- Style Guide for Python Code](https://www.python.org/dev/peps/pep-0008/#tabs-or-spaces)
[^6]: [Ruby Style Guide](https://rubystyle.guide/#spaces-indentation)
