---
title: "論理削除で破壊する, RDBの整合性"
emoji: "🔨"
type: "tech"
topics: ["SQL"]
published: true
---

# 削除とは

> 削除（さくじょ）とは、一度作成された文書やデータなどを削って取り除くこと。[^1]

データの削除には次の二種類があります.

- この世から存在自体を無かったものにする
- ユーザが利用可能な領域から取り除く
  (隠蔽するのみ. システムからは参照可能な状態を維持する)

このうちの前者の「完全に削除する」と言っているのが**物理削除**です.
対して後者の「削除はするが削除はしない」というような訳の分からんことを言っているのが**論理削除**です.

この記事は**論理削除が如何に悪徳な手法であるのか**を語る場です.

以下で記述している SQL や Procedure は PostgreSQL, PL/pgSQL の想定で記述されています.



## 論理削除の例

「削除した場合には削除の代わりにフラグを立て, ユーザから見えないようにする」というのが典型的な形態の論理削除になります.

例えば, 次ようなユーザテーブルがあった場合を考えます.
```sql
CREATE TABLE users (
  id SERIAL NOT NULL PRIMARY KEY,
  name VARCHAR NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE,
  password_digest VARCHAR(255) NOT NULL
);
```

ここに更に`deleted_at`や`is_deleted`といったカラムを追加し.

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP;
```

「DELETE 文の発行の代わりに`deleted_at`カラムを削除時刻で更新するようにしよう!」と決めます.

```sql
-- 物理削除
DELETE FROM users WHERE id = 1

-- 論理削除
UPDATE users SET deleted_at = NOW() WHERE id = 1
```

このようにした場合, ユーザから見える範囲に`deleted_at IS NULL`の条件を付けることで, ユーザにはあたかも削除されたかのように見せつつもデータを保持し続けることができます.



# 論理削除によるメリットに見えるもの

論理削除に行き着いてしまう理由は, 次のようなケースがあります.

- 誤ってしまってもすぐ復元できる
- 後から復元できるようにデータを残しておくことが出来る
- 記録として残しておき, 後々の調査や分析に役立てることが出来る


# 論理削除によるデメリット

論理削除は「削除されたはずのデータを残す」というものです.
これは「無矛盾性があり」「整合的であり続ける」ということを目指す RDB のデータ保持においては致命的な問題を引き起こします.




# 問題ケース

## 1. サボりだす外部キー制約

外部キー制約は**参照先に適切なデータが存在しない場合はデータの作成を許容しない**という一種の制約事項です.
そして同時に**削除や更新が実行されたデータと, そのデータ参照しているデータの挙動を定義する**という一種の関連宣言でもあります.

前述の`users`テーブルに加えて, 次の`posts`テーブルを考えます.
```sql
CREATE TABLE posts (
  id SERIAL NOT NULL PRIMARY KEY,
  author_id INTEGER NOT NULL,
  title VARCHAR(255) NOT NULL,
  body TEXT NOT NULL
);

-- create index
CREATE INDEX idx_posts_user ON posts ( author_id );

-- add foreign key
ALTER TABLE posts 
  ADD CONSTRAINT fk_posts_user
  FOREIGN KEY (author_id)
    REFERENCES users (id)
    ON DELETE CASCADE
    ON UPDATE CASCADE;
```

このテーブル構成は, 定義からして次のことを要請しています.

- 実在するユーザが作成した投稿のみを受け付け, 保持すること
- ユーザが削除された場合は投稿も同様に削除すること

### 物理削除の場合

ユーザを削除した後にもう一度同一のユーザで記事を作成しようとすると, 物理削除の場合はエラーを吐いて矛盾したデータが構築されるのを防いでくれます. とてもいい子です.
```sql
CREATE OR REPLACE FUNCTION create_post_test() RETURNS boolean AS $$
DECLARE
  user_id INTEGER;
BEGIN
  -- ユーザ作成
  INSERT INTO users 
    (name, email, password_digest) 
    VALUES 
    ('foobar', 'sample@example.com', 'xxxxxxxxx') RETURNING id INTO user_id;
  -- 存在しているユーザに対する投稿の登録は許容される
  INSERT INTO posts
    (author_id, title, body)
    VALUES
    (user_id, 'サンプル記事', 'サンプルです');
  
  -- 物理削除 (+ 記事が自動的に削除される)
  DELETE FROM users WHERE id = user_id;

  -- 削除済みユーザに対する投稿の登録は許可されない
  INSERT INTO posts
    (author_id, title, body)
    VALUES
    (user_id, 'サンプル記事', 'サンプルです');

  RETURN TRUE;
END;
$$ LANGUAGE plpgsql;
```

```sql
SELECT create_post_test();
-- ERROR:  insert or update on table "posts" violates foreign key constraint "fk_posts_user"
-- DETAIL:  Key (author_id)=(1) is not present in table "users".
-- CONTEXT:  SQL statement "INSERT INTO posts
--   (author_id, title, body)
--   VALUES
--   (user_id, 'サンプル記事', 'サンプルです')"
-- PL/pgSQL function create_post_test() line 20 at SQL statement

SELECT * FROM posts;
-- no data
```

### 論理削除の場合

この外部キー制約は効力を成さず, **本来削除されている, 存在していないはずのユーザからの投稿を受け付けます**.
更には**DELETE CASCADEによる制御がなされないため, 存在しなくなったユーザの記事を保持し続けます**.
同様の関数を実行してみましょう. 結果は次のようになるはずです.

```sql
CREATE OR REPLACE FUNCTION create_post_test() RETURNS boolean AS $$
DECLARE
  user_id INTEGER;
BEGIN
  -- ユーザ作成
  INSERT INTO users 
    (name, email, password_digest) 
    VALUES 
    ('foobar', 'sample@example.com', 'xxxxxxxxx') RETURNING id INTO user_id;
  -- 存在しているユーザに対する投稿の登録は許容される
  INSERT INTO posts
    (author_id, title, body)
    VALUES
    (user_id, 'サンプル記事', 'サンプルです');
  
  -- 論理削除
  UPDATE users SET deleted_at = NOW() WHERE id = user_id;

  -- 削除済みユーザのはずだが許可される
  INSERT INTO posts
    (author_id, title, body)
    VALUES
    (user_id, 'サンプル記事', 'サンプルです');

  RETURN TRUE;
END;
$$ LANGUAGE plpgsql;
```

```sql
SELECT create_post_test();
-- t

-- 削除済みユーザの記事が残存し, 追加で作成されている
SELECT * FROM posts;
-- 1 |         1 | サンプル記事 | サンプルです
-- 2 |         1 | サンプル記事 | サンプルです
```

schema 定義は「許容しない」「連動して消える」のはずが, 真逆の「許容する」「消えないし後から追加出来る」というものになります.
**これは論理削除が引き起こした不整合です**.

### ちょこちょこ聞く言い訳
- 「連動して消えるレコードも更新するに決まってるだろ!」
  - 後から追加されたレコードは削除された状態とはなりません.
- 「追加する際にも削除されているのかどうかを判定をするに決まってるだろ!」
  - 複数の別 Transaction で「ユーザの削除」と「投稿の作成」が並列実行された場合に矛盾が生じます.
- 「Trigger でチェック/処理すればいいじゃん」
  - 外部キー毎に Trigger 書くとしたら外部キー貼った意味は何でしょうか?
- 「削除済みユーザでもデータ持ちたい時あるじゃん」
  - であれば「退会済みユーザ」というような形態でデータを持つよう設計すべきです.
- 「外部キーなんて要らないよ」
  - うちのチームには君なんて要らないよ

## 2. 過剰に働くUNIQUE制約

UNIQUE 制約は, **指定したカラムについて, 重複しているレコードの作成を許容しない**という一種の制約です.
前述の`users`テーブルには`email`カラムに UNIQUE 指定が記述されています.

```sql
CREATE TABLE users (
  id SERIAL NOT NULL PRIMARY KEY,
  name VARCHAR NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE,
  password_digest VARCHAR(255) NOT NULL
);
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP;
```

ここで「一度ユーザがサービスを利用するのを止めてから再度サービスを利用開始した」というケースを考えてみます.

### 物理削除の場合

特に問題は発生しません.

```sql
CREATE OR REPLACE FUNCTION create_user_twice_test() RETURNS boolean AS $$
BEGIN
  -- ユーザ作成
  INSERT INTO users 
    (name, email, password_digest) 
    VALUES 
    ('foobar', 'sample@example.com', 'xxxxxxxxx');
  
  -- 物理削除 
  DELETE FROM users WHERE id = LASTVAL();

  -- ユーザ再作成 (重複なし) 成功
  INSERT INTO users 
    (name, email, password_digest) 
    VALUES 
    ('foobar', 'sample@example.com', 'xxxxxxxxx');

  RETURN TRUE;
END;
$$ LANGUAGE plpgsql;
```

```sql
SELECT create_user_twice_test();
-- t
```

### 論理削除の場合

**過去に存在していたユーザを参照してUNIQUE制約違反エラーが出ます**.

```sql
CREATE OR REPLACE FUNCTION create_user_twice_test() RETURNS boolean AS $$
BEGIN
  -- ユーザ作成
  INSERT INTO users 
    (name, email, password_digest) 
    VALUES 
    ('foobar', 'sample@example.com', 'xxxxxxxxx');
  
  -- 論理削除 
  UPDATE users SET deleted_at = NOW() WHERE id = LASTVAL();

  -- ユーザ再作成 (重複) 失敗
  INSERT INTO users 
    (name, email, password_digest) 
    VALUES 
    ('foobar', 'sample@example.com', 'xxxxxxxxx');

  RETURN TRUE;
END;
$$ LANGUAGE plpgsql;
```

```sql
SELECT create_user_twice_test();
-- ERROR:  duplicate key value violates unique constraint "users_email_key"
-- DETAIL:  Key (email)=(sample@example.com) already exists.
-- CONTEXT:  SQL statement "INSERT INTO users 
--     (name, email, password_digest) 
--     VALUES 
--     ('foobar', 'sample@example.com', 'xxxxxxxxx')"
-- PL/pgSQL function create_user_twice_test() line 4 at SQL statement
```

これも schema 定義は「存在している物は一意とする. かつ存在しない物は登録可」でした.
しかし実情は「過去あった物と存在している物を合わせて一意にならない物は登録不可」というものになります.
**これも論理削除が引き起こした不整合です**.

### ちょこちょこ聞く言い訳
- 「`deleted_at`との複合 UNIQUE 制約で良くなるよ」
  - UNIQUE 制約で NULL は重複とはみなされないので, それは「削除済みの中で UNIQUE」という全く意味のないものになります
- 「`deleted_at`の真逆(存在している時 true, 削除時 NULL)のカラムを更に追加して, そのカラムとの複合 UNIQUE 制約で良くなるよ」
  - 同じ意味のカラムを複数持つ必要性が皆無です. 正規化って知ってる?
- 「UNIQUE がはられてるカラムは削除時にランダム値を入れるようにしよう」
  - 初期の「復元できるように」とか「記録として」みたいな概念どこ行きました?
- 「UNIQUE 制約なんて外して良くない?」
  - きみなんてチームから外して良くない?

### 多少妥当な対応
- 「部分 index を使った UNIQUE 制約で良くなるよ」
  - かしこいね. 部分 index 対応 RDBMS ならね. MySQL だと無理だね.
- 「`deleted_at`の真逆(存在している時 true, 削除時 NULL)のカラムを`deleted_at`の代わりに足して, そのカラムとの複合 UNIQUE 制約で良くなるよ」
  - カーディナリティ 2 のカラムを含む index による性能悪化がないことを天に祈ろう.

## 3. 残り続けるデータ, 増え続けるデータ

論理削除にするというのは, 過去にあった**本来削除されているデータも全て残す**という選択です.

そのため, 物理削除は「現在使われているリソース」で済むのに対して, 論理削除は「過去累積の総リソース」を処理し続けることが要求されます.

単純にこれは次のような問題を引き起こします.
- データを格納するためのストレージの費用の増大
- データ量による SQL 検索速度の劣化, 負荷の上昇

これが**そんな大したことないでしょ**と軽く捉えられているのが最たる問題点だと認識しています.

とても単純に「毎日新規ユーザが 1 人入り, 各々 1 日の終わりに 1% の確率で退会し, 各ユーザは毎日 1MB ほどのデータを蓄積する」という場合をグラフ化してみましょう.
その容量の遷移が明確に見えますが, 「論理削除にした」というただ一点の汚点のため故に, データの蓄積は止まらず, 物理削除の場合に比べて著しい量のデータを溜め込むことになります.

![](/images/logical_delete_dystopia/data_amount.png)

これは不整合を引き起こすようなものではありませんが, 確実に運営していくにあたって性能的な課題点, 費用的な障壁となって立ちはだかります.


### ちょこちょこ聞く言い訳

- ログファイル残しとくのと同じようなものだよ
  - ログファイルは圧縮処理やローテーションが効きます. DB をどう処理するおつもりで?
- 不要になったデータは随時物理削除すればいいんだよ
  - 「不要になった」の判断基準はなんですか...??? なんで残したんですか...??? はじめから物理削除すればよかったのでは?

なお前述の通り外部キー制約等が正常に制御されなくなるため, **論理削除されているデータがアクティブなデータに紐付いていない保証はどこにもありません**.

# どうするべきなのか

## 復元可能なようにしておきたい!

データ復元したい場面を考えると, 想起されるのは「障害に伴うデータロストからの復旧」「構成変更等によって生じた削除の取り消し」といった物になります.
これらに対しては論理削除は無力です. データロストしても論理削除されているから大丈夫☆となることはまずありません. 
これらの場面では**論理削除後のデータ自体も消えている**というケースが殆どになります.
なにせ, 復旧したいデータと, 全く同じ場所で, 全く同じ機構の中に存在しているのですから.

復元についての代替案として最も簡素なものは, **データバックアップの作成** になります.
もし障害によりデータベースが丸ごと吹き飛んだとしても, バックアップデータを別途保管できていればそこからリストアできます.
大規模な構成変更の際には事前にバックアップを取得することで, 実施に際してなんらかのトラブルが発生した場合も, 変更前の状態に復元できます.
他にも FailOver の用意等で障害そのものへの対抗措置を取っておく等でそもそも復元が不要にしておくことも有用でしょう.

## 記録として残しておいて分析調査をしたい!

まずユーザの情報を分析に用いてよいのか? 分析のため実データにアクセスして良いのか? というコンプライアンス的な問題があります.
利用規約等で同意を取っており問題ない場合であっても, 性能面に支障をきたします.
レコード数が物理的に増加することになるためシステム負荷は上がりやすくなります. また実データを保管しているサーバに負荷をかけて分析クエリを実行させるような構成は, 分析の度にユーザに対する負荷影響が懸念されナンセンスです.

分析を行いたいのであれば, 別途**分析用のデータストア**を用意してデータを蓄積していく方が望ましくあります.
実レコードを残すのではなく, 特定のイベントの発生を DWH 等に蓄積していけば, レコードが物理削除されたとしても過去情報の分析は可能です. 
生レコードをそのまま触れるわけではないため, ユーザの個人情報等のデータマスクも実施できるでしょう.
RDB 自体のレコード数が増加するわけでもないため, 処理負荷の増大も回避することが出来ます.

# まとめ

論理削除は次のような問題点を有するメソッドです.

- RDB の無矛盾性を破壊する
- 本来可能だったデータの整合性確保を不可能にする
- サービスの性能を劣化させる
- サーバ費用を増大させる

デメリットを認識し, それでもなお利用する明確な目的が存在する方以外は利用しないようにしましょう.
また, 論理削除しようとしている方がいらっしゃったら, 優しく「メリットは何?」と理詰めしてあげましょう.
サービスの未来を守るためにも.


[^1]: [削除-wikipedia](https://ja.wikipedia.org/wiki/%E5%89%8A%E9%99%A4)