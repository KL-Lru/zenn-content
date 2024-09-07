---
title: "Inbound Email Parse Webhook を Rails で受け取ってたら500エラー祭りにあった"
emoji: "🚇"
type: "tech"
topics: ["Rails", "SendGrid"]
published: true
---

## はじめに

SendGrid は様々なところで利用されている, メール配信サービスの一つです.
SendGrid にはメール受信を行うための Inbound Parse Webhook という機能があります.

https://sendgrid.kke.co.jp/docs/API_Reference/Webhooks/parse.html

この Webhook を Rails で受け取るようなものを運用していたのですが, これが 8 月末辺りから様子がおかしくなり, 500 エラーを連発するようになっていました.
その原因調査と対応の備忘録です.

## TL;DR

次の条件を全て満たすリクエストが来た場合に強制的に 500 エラーが返却されます.

- `multipart/mixed`などの Multipart リクエスト
- `charset`が`ISO-2022-JP`や`UTF-7`などのテキストフィールドが含まれている

Inbound Parse Webhook は ISO-2022-JP でエンコードされたメールを受信した場合, この条件に該当するリクエストを送信してくるようになったため, 500 エラーが発生していました.

先人が PR を立ててくれているので, Rack の`Rack::Multipart::Parser`にパッチを当てよう.

https://github.com/rack/rack/pull/2247

#### 記述コードの前提

解説用に Rails アプリケーション内に Webhook を受け取る以下のような Controller が用意されているものとして記述します.

```ruby:app/controllers/webhooks_controller.rb
class WebhooksController < ApplicationController
  protect_from_forgery with: :null_session

  def parse
    Rails.logger.info(params)

    head :ok
  end
end
```

```ruby:config/routes.rb
Rails.application.routes.draw do
  post "" => "webhooks#parse"
end
```

## Rack での文字エンコーディング指定時の挙動

結論から言えば Rack の Multipart リクエストの解析処理部分でエラーが発生していました.

### UTF-8

次のような curl コマンドで送信されたリクエストを考えてみます.

```sh
curl --trace-ascii request.log -X POST -H "Content-Type: multipart/mixed" -F "text=あいうえお" http://localhost:3000/
```

この場合は問題なく Rails までリクエストが到達し, 処理されます.

```sh
Processing by WebhooksController#parse as */*
  Parameters: {"text"=>"あいうえお"}
Can't verify CSRF token authenticity.
#<ActionController::Parameters {"text"=>"あいうえお", "controller"=>"webhooks", "action"=>"parse"} permitted: false>
Completed 200 OK in 0ms (ActiveRecord: 0.0ms | Allocations: 121)
```

### ISO-2022-JP

日本のメールでよく使われている文字エンコードとして ISO-2022-JP があります.
メールで利用されることを前提としているため, 7bit のコードになっていることが特徴です.
現代では Gmail 等主要なメーラーは概ね UTF-8 を利用している印象ですが, 今も根強く使われているエンコードです.

SendGrid の Inbound Parse Webhook はメールが ISO-2022-JP でエンコードされている場合は, そのエンコードに従ってリクエストを送信してきます.

```sh
value=$(echo "あいうえお" | iconv -f UTF-8 -t ISO-2022-JP)
curl --trace-ascii request.log -X POST -H "Content-Type: multipart/mixed" -F "text=${value};type=text/plain; charset=iso-2022-jp" http://localhost:3000
```

このエンコードで送付した場合, 応答として 500 エラーが返却され, ログ上には Puma でエラーを処理した旨と, そのエラー内容が表示されます.

#### Rails 7.1 以降(Rack 3 系)を利用している場合

```
2024-09-07 17:28:32 +0900 Rack app ("POST /" - (127.0.0.1)): #<Encoding::CompatibilityError: incompatible character encodings: ISO-2022-JP and UTF-8>
```

:::details StackTrace

```
Puma caught this error: incompatible character encodings: ISO-2022-JP and UTF-8 (Encoding::CompatibilityError)
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-3.1.7/lib/rack/query_parser.rb:106:in `index'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-3.1.7/lib/rack/query_parser.rb:106:in `_normalize_params'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-3.1.7/lib/rack/query_parser.rb:95:in `normalize_params'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-3.1.7/lib/rack/request.rb:688:in `block in expand_param_pairs'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-3.1.7/lib/rack/request.rb:687:in `each'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-3.1.7/lib/rack/request.rb:687:in `expand_param_pairs'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-3.1.7/lib/rack/request.rb:529:in `POST'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-3.1.7/lib/rack/method_override.rb:49:in `method_override_param'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-3.1.7/lib/rack/method_override.rb:33:in `method_override'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-3.1.7/lib/rack/method_override.rb:21:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-3.1.7/lib/rack/runtime.rb:24:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/activesupport-7.1.4/lib/active_support/cache/strategy/local_cache_middleware.rb:29:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/actionpack-7.1.4/lib/action_dispatch/middleware/server_timing.rb:59:in `block in call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/actionpack-7.1.4/lib/action_dispatch/middleware/server_timing.rb:24:in `collect_events'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/actionpack-7.1.4/lib/action_dispatch/middleware/server_timing.rb:58:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/actionpack-7.1.4/lib/action_dispatch/middleware/executor.rb:14:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/actionpack-7.1.4/lib/action_dispatch/middleware/static.rb:25:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-3.1.7/lib/rack/sendfile.rb:114:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/actionpack-7.1.4/lib/action_dispatch/middleware/host_authorization.rb:141:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/railties-7.1.4/lib/rails/engine.rb:536:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/puma-6.4.2/lib/puma/configuration.rb:272:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/puma-6.4.2/lib/puma/request.rb:100:in `block in handle_request'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/puma-6.4.2/lib/puma/thread_pool.rb:378:in `with_force_shutdown'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/puma-6.4.2/lib/puma/request.rb:99:in `handle_request'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/puma-6.4.2/lib/puma/server.rb:464:in `process_client'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/puma-6.4.2/lib/puma/server.rb:245:in `block in run'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/puma-6.4.2/lib/puma/thread_pool.rb:155:in `block in spawn_thread'
```

:::

#### Rails 7.0 以前(Rack 2 系)を利用している場合

```
2024-09-07 17:47:25 +0900 Rack app ("POST /" - (127.0.0.1)): #<Encoding::CompatibilityError: incompatible encoding regexp match (US-ASCII regexp with ISO-2022-JP string)>
```

:::details StackTrace

```
Puma caught this error: incompatible encoding regexp match (US-ASCII regexp with ISO-2022-JP string) (Encoding::CompatibilityError)
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/query_parser.rb:90:in `normalize_params'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/multipart/parser.rb:209:in `block (2 levels) in result'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/multipart/parser.rb:107:in `get_data'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/multipart/parser.rb:207:in `block in result'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/multipart/parser.rb:130:in `block in each'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/multipart/parser.rb:130:in `each'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/multipart/parser.rb:130:in `each'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/multipart/parser.rb:206:in `result'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/multipart/parser.rb:84:in `parse'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/multipart.rb:53:in `extract_multipart'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/request.rb:594:in `parse_multipart'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/request.rb:446:in `POST'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/method_override.rb:45:in `method_override_param'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/method_override.rb:29:in `method_override'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/method_override.rb:17:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/runtime.rb:22:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/activesupport-7.0.8.4/lib/active_support/cache/strategy/local_cache_middleware.rb:29:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/actionpack-7.0.8.4/lib/action_dispatch/middleware/server_timing.rb:61:in `block in call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/actionpack-7.0.8.4/lib/action_dispatch/middleware/server_timing.rb:26:in `collect_events'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/actionpack-7.0.8.4/lib/action_dispatch/middleware/server_timing.rb:60:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/actionpack-7.0.8.4/lib/action_dispatch/middleware/executor.rb:14:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/actionpack-7.0.8.4/lib/action_dispatch/middleware/static.rb:23:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/rack-2.2.9/lib/rack/sendfile.rb:110:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/actionpack-7.0.8.4/lib/action_dispatch/middleware/host_authorization.rb:138:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/railties-7.0.8.4/lib/rails/engine.rb:530:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/puma-5.6.8/lib/puma/configuration.rb:252:in `call'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/puma-5.6.8/lib/puma/request.rb:77:in `block in handle_request'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/puma-5.6.8/lib/puma/thread_pool.rb:340:in `with_force_shutdown'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/puma-5.6.8/lib/puma/request.rb:76:in `handle_request'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/puma-5.6.8/lib/puma/server.rb:443:in `process_client'
/home/lru/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0/gems/puma-5.6.8/lib/puma/thread_pool.rb:147:in `block in spawn_thread'
```

:::

## 対応方法

数日前に先人が Rack に対してすでにプルリクエストを構築してくれているため, 特に問題がクリティカルではないならライブラリのアップデートを待つのも一つの手です.
今すぐの修正が必要な場合は, Rack の Multipart リクエストの解析処理部分に対して, モンキーパッチを当ててやることでエラーなくリクエストを処理できるようになります.

ISO-2022-JP 以外に UTF-7 などへの対応も必要な場合, エンコードを戻している処理部分を適宜修正してください.
また, 非 UTF-8 を含む POST リクエストであるため, Rails6.1 以降に Controller でエンコーディング対処されていた方々は, ISO-2022-JP でエンコードされたリクエストだけ二重デコードしてしまう可能性があります.
Controller 側を修正するか, このパッチ内では`encode`の代わりに`force_encoding`を使うなどの対応を行うよう注意してください.

### Rack 3 系向け

```ruby:config/initializers/patches/rack_multipart_parser_patch.rb
if defined?(Rack::Multipart::Parser)
  raise "Please reconfirm the need for patching to Rack::Multipart::Parser" if Rack::RELEASE != "3.1.7"

  module RackMultipartParserPatch
    def result
      @collector.each do |part|
        part.get_data do |data|
          tag_multipart_encoding(part.filename, part.content_type, part.name, data)
          name, data = handle_dummy_encoding(part.name, data)
          @query_parser.normalize_params(@params, name, data)
        end
      end
      Rack::Multipart::Parser::MultipartInfo.new @params.to_params_hash, @collector.find_all(&:file?).map(&:body)
    end

    def handle_dummy_encoding(name, body)
      # 'dummy' encodings are not properly implemented in Ruby MRI
      if name.encoding.dummy?
        if name.encoding == Encoding::ISO_2022_JP
          name = name.encode(Encoding::UTF_8)
          body = body.encode(Encoding::UTF_8)
        end
      end
      return name, body
    end
  end

  Rack::Multipart::Parser.prepend(RackMultipartParserPatch)
end
```

### Rack 2 系向け

```ruby:config/initializers/patches/rack_multipart_parser_patch.rb
if defined?(Rack::Multipart::Parser)
  raise "Please reconfirm the need for patching to Rack::Multipart::Parser" if Rack::RELEASE != "2.2.9"

  module RackMultipartParserPatch
    def result
      @collector.each do |part|
        part.get_data do |data|
          tag_multipart_encoding(part.filename, part.content_type, part.name, data)
          name, data = handle_dummy_encoding(part.name, data)
          @query_parser.normalize_params(@params, name, data, @query_parser.param_depth_limit)
        end
      end
      Rack::Multipart::Parser::MultipartInfo.new @params.to_params_hash, @collector.find_all(&:file?).map(&:body)
    end

    def handle_dummy_encoding(name, body)
      # 'dummy' encodings are not properly implemented in Ruby MRI
      if name.encoding.dummy?
        # ISO-2022-JP is a legacy but still widely used encoding in Japan
        # Here we properly convert ISO-2022-JP to UTF-8
        if name.encoding == Encoding::ISO_2022_JP
          name = name.encode(Encoding::UTF_8)
          body = body.encode(Encoding::UTF_8)
        end
      end
      return name, body
    end
  end

  Rack::Multipart::Parser.prepend(RackMultipartParserPatch)
end
```

このパッチの適用で, ISO-2022-JP でエンコードされたリクエストを正常に処理できるようになります.

```sh
value=$(echo "あいうえお" | iconv -f UTF-8 -t ISO-2022-JP)
curl --trace-ascii request.log -X POST -H "Content-Type: multipart/mixed" -F "text=${value};type=text/plain; charset=iso-2022-jp" http://localhost:3000
```

```sh
Processing by WebhooksController#parse as */*
  Parameters: {"text"=>"あいうえお"}
Can't verify CSRF token authenticity.
#<ActionController::Parameters {"text"=>"あいうえお", "controller"=>"webhooks", "action"=>"parse"} permitted: false>
Completed 200 OK in 1ms (ActiveRecord: 0.0ms | Allocations: 1096)
```

## 原因調査

### エラーの根源は?

Rack のスタックトレースから見ると, `Rack::QueryParser`の`normalize_params`メソッド内でエラーが発生していることがわかります.
私が使っていたのは Rack 2 系で, こちらで調査してしまったため, こちらについて記述します.

https://github.com/rack/rack/blob/b1deebdc0a4f61cc141cece5a911917ff1e4b901/lib/rack/query_parser.rb#L87-L100

エラー箇所は 90 行目の`name`と正規表現でマッチングを行っている箇所です.
この部分で`Encoding::CompatibilityError`が発生しています. マッチングを取っている正規表現は`UTF-8`や`US-ASCII`であり, これと`ISO-2022-JP`の文字列をマッチングさせることができないエラーとなっているため, 問題は`name`のエンコーディングが`ISO-2022-JP`であることにあります.

このメソッドを呼び出している箇所は`Rack::Multipart::Parser`の`result`メソッドです.

https://github.com/rack/rack/blob/b1deebdc0a4f61cc141cece5a911917ff1e4b901/lib/rack/multipart/parser.rb#L205-L213

209 行目で`normalize_params`を呼び出しています
そしてその前にある`tag_multipart_encoding`メソッドを読むと, `name`のエンコーディングを変更していることがわかります.

https://github.com/rack/rack/blob/b1deebdc0a4f61cc141cece5a911917ff1e4b901/lib/rack/multipart/parser.rb#L349-L375

このメソッドでは Multipart の各フィールドについて, `Content-Type`の指定を解析し, `;`の後に続く`charset`の指定があればそれをもとにエンコーディングを変更しています.

ということで「リクエスト中のフィールドに`charset`指定が`ISO-2022-JP`などの Ruby 内で UTF-8 の正規表現とマッチングできないエンコーディングのものが存在している」が原因だったということはわかります.
そしてこのメソッド中にある通り, ファイルフィールドについてはエンコーディングの識別/変更処理を通過しないため, 「テキストフィールドに限る」ということもわかります.

このエンコーディング処理が原因でエラーが発生しているため, `tag_multipart_encoding`メソッドでエンコーディングが変更された後に, 処理可能なものに戻してやればエラーは解消されるはずです. ということで対応方法については前述の通りです.

### なぜ先月末から?

残念ながら私の観測できる範囲ではリクエスト受け取り時の生リクエストボディまでログに取っている箇所はなかったため推測しかできません.
エラー根源からして `charset=iso-2022-jp` などがきちんと指定されるように改修が入った, というのが一番可能性が高いと思っています.

```sh
--------------------------a522f7192cb8ee28
Content-Disposition: attachment; name="text"
Content-Type: text/plain; charset=iso-2022-jp

.$B$"$$$&$($*.(B
--------------------------a522f7192cb8ee28--
```

なおこの Charset 指定は`charset`の指定として問題ない値が指定されている限りは当然何も問題のあるリクエストではありません. ただ Rack の Parser が対応できていなかったというだけです.

## おわりに

文字コードの闇, 深すぎんか?
