---
title: "最低賃金を出してくれるAPIをノリで作った"
emoji: "💸"
type: "tech"
topics: ["Python", "FastAPI", "Vercel"]
published: true
---

# TL;DR

https://minimum-wage-api.vercel.app/docs

https://github.com/LruLab/minimum-wage-api

これを勢いで作った話.

## 始まり

「求人出すときとか, 既に出してる求人更新するときに最低賃金を毎回確認するの面倒じゃない?」という話がふわふわと聞こえてきました.
そもそもこのご時世に最低賃金レベルで正規雇用の求人を出すなというのはさておき, 確かに毎回確認するのは単純に手間です.

じゃあ早い話, 自動チェッカーを作ればいいじゃない, と言いたいところですが厚労省は API を出してくれていないので自動チェックできない.
というわけで, さくっと最低賃金を出してくれる API を作ることにしました.

## 先行研究

https://takaya1992.hatenablog.jp/entry/2017/04/06/022344

どうやら同じような始まりで最低賃金 API を作っている人がいらっしゃいました. すごい.
が, 6 年間更新がなく, 現在利用するのは厳しい状態にあります.

## 使ったもの

| 区分            | 対象                                               |
| :-------------- | :------------------------------------------------- |
| Web Framework   | [FastAPI](https://fastapi.tiangolo.com/)           |
| Data Management | [TinyDB](https://tinydb.readthedocs.io/en/latest/) |
| API Document    | [ReDoc](https://redocly.com/)                      |
| Hosting         | [Vercel](https://vercel.com/)                      |

### データの管理

特に RDB を使う必要のないデータではないので, TinyDB を用いて JSON ファイルで管理します.
JSON であるため, 常にどんなデータが含まれているのかはリポジトリを見れば確認でき, 更新時の差分取得も容易です.
できれば気軽に更新 PR とか投げてほしい.

### データの取得方法

適用が開始される日時のデータを保持し, 日時に応じてその時々の最低賃金を取得可能にします.
デフォルトとして現在時刻の最低賃金を取得することで「事前に更新後の最低賃金を登録しておき, 日付が変わった時点で対象の賃金が切り替わる」ということを実現しています.

### ホスティング

API として提供するため, 「ドキュメントの置き場」と「API の置き場」の両方を用意する必要がありました.
Vercel はパス毎に異なるサービスをまとめてデプロイできるので, それを利用して API とドキュメントをまとめてデプロイしています.

## 所感

Vercel は初めて使ったのですが, 複数の公開対象をまとめてデプロイできる上, ドメインもいい感じに確保されてくれるので結構体感良かったです.
逆に公開しないものの設定が必要になってくる点はちょっとめんどうでした. (はじめ requirements.txt 的な物まで公開されてしまっていて routing 設定調整しました)

API ドキュメントは ReDoc 利用すると Swagger よりちょっと見た目が良くなるのも良きでした.

### 苦労点

過去のデータも含め登録したのですが, PDF からのデータ取得がちょっとだけ面倒でした.
`pdfminer`などを利用して抽出自体はできますが, たまになんで? となるものがあります.

```python
from pdfminer.high_level import extract_text
from urllib.request import urlretrieve
from pprint import pprint

urlretrieve("https://www.mhlw.go.jp/content/11200000/001140686.pdf", "tmp.pdf")
text = extract_text("tmp.pdf")
pprint(text)
```

3 文字以上の都道府県の表記が何故か逆順になり...

```python
 ...
 '北 海 道\n'
 '森\n'
 '青\n'
 '手\n'
 '岩\n'
 '城\n'
 '宮\n'
 '田\n'
 '秋\n'
 '形\n'
 '山\n'
 ...
```

年月日いづれかが 1 文字だと微妙な位置にスペースが入ったり... (平成令和変換も必要だし...)

```python
 ...
 '平 成 28 年 10 月 1 日\n'
 '平成28年10月20日\n'
 '平 成 28 年 10 月 5 日\n'
 '平 成 28 年 10 月 5 日\n'
 '平 成 28 年 10 月 6 日\n'
 '平 成 28 年 10 月 7 日\n'
 '平 成 28 年 10 月 1 日\n'
 ...
```

もう CSV とかにして配布してくれないかな...

### 今後あるかも...?

自動更新はありません. だって日本だもの. 絶対 HTML 変わるじゃん.
来年も構造が変わっていなければ作る可能性はゼロではないです.

## 参考

https://www.mhlw.go.jp/stf/seisakunitsuite/bunya/koyou_roudou/roudoukijun/minimumichiran/
