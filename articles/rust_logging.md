---
title: "実装して学ぶ, Rust の `log` Crate"
emoji: "🦀"
type: "tech"
topics: ["Rust", "Log"]
published: true
---

## この記事について

本記事においては, Rust 公式の `log` Crate を利用したロギングについて語ります.
今回はあえて Logger 実装ライブラリには触れず, 独自実装する場合について解説します.

https://github.com/rust-lang/log

## Log の重要性

アプリケーション / サービスにおいて, ログ情報は重要な役割を果たします.

- システムの稼働状態を把握するための情報源
- システムのパフォーマンスを計測するための情報源
- エラーの原因を特定するための情報源 etc...

これらの情報を出力するにあたって, 効率的に情報を利用するために考慮すべき点がいくつもあります.

- ログの出力先は標準出力にするか, ファイルにするか, 通信して外部書き出すか?
- ログの出力形式は JSON にするか, プレーンテキストにするか?
- ログの出力レベルはどのように設定するか?

これらの制御をするにあたって, 大体の言語 / Framework でログ出力機能が提供されています.

Rust においては, この`log` Crate がデファクトなログ出力機構となっています.

## `log` Crate の利用方法

シンプルにトレースログなら `trace!`, 情報ログなら `info!` マクロを利用して記述できます.

```rust
use log::{info, trace, warn};

pub fn fizz_buzz(num: i32) {
    trace!("Starting fizz_buzz with num: {}", num);
    if num < 1 {
        warn!("Invalid input: {}", num);
        return;
    }

    for i in 1..num + 1 {
        match i {
            i if i % 15 == 0 => info!("FizzBuzz"),
            i if i % 3 == 0 => info!("Fizz"),
            i if i % 5 == 0 => info!("Buzz"),
            _ => info!("{}", i),
        }
    }
}
```

この Crate の素晴らしいところは,ログを出力するためのマクロと Interface のみを提供するファサードとなっている点です.
ログの出力先, フォーマットについてはすべて `Log` Trait を実装した Logger の実装に依存するものとなります.
デフォルトでは「何も出力しない」という Logger が設定されているため, 何も出力されません.

この `Log` Trait の実装を提供するライブラリが各所で展開されており, 代表的なものの一覧がリポジトリ上に記載されています.

https://github.com/rust-lang/log#in-executables

## `Log` Trait の実装

ログを出力するために必要となる `Log` Trait で, 要求されるメソッド[^1]は次の 3 つです.

```rust
pub trait Log: Sync + Send {
    /// 特定のメタデータのログを出力するかどうかを判定する関数
    fn enabled(&self, metadata: &Metadata) -> bool;

    /// 実際にログを出力する関数
    fn log(&self, record: &Record);

    /// bufferに溜まっているログをflushする関数
    fn flush(&self);
}
```

### `enabled`

メタデータを受け取り, ログを出力するかどうかを判定するメソッドです.
メタデータには今出力しようとしているログのログレベル, ログ出力ターゲットとして設定されている文字列情報が含まれています.

例えばこれを利用すると, 特定ターゲットのログのみを出力するよう制御できます.

```rust
struct JsonLogger {};

impl log::Log for JsonLogger {
    fn enabled(&self, meta: &log::Metadata) -> bool {
        meta.target() == "json_log"
    }

    fn log(&self, record: &log::Record) {
        if self.enabled(record.metadata()) {
            println!(
                r#"{{"severity":"{}","message":"{}"}}"#,
                record.level(),
                record.args()
            );
        }
    }

    fn flush(&self) {}
}
```

この Logger は, `json_log` というターゲットの設定されたログ以外のログをすべて無視します.

```rust
info!(target: "json_log", "Hello, world!");
// => {"severity":"INFO","message":"Hello, world!"}
info!(target: "std_log", "Hello, world!"); // ignored
info!("Hello, world!"); // ignored
```

他にもパターンとしては次のようなものがあります.

- ログレベルが `Error` 以上の場合のみに対応したフォーマットで出力する

```rust
fn enabled(&self, meta: &log::Metadata) -> bool {
    meta.level() <= log::Level::Error
}
```

- 環境変数を参照して, 特定のレベル以上のみ出力するようを切り替える

```rust
fn enabled(&self, meta: &log::Metadata) -> bool {
    let env_level = std::env::var("LOG_LEVEL").unwrap_or("info".to_string());
    let level = env_level.parse::<log::Level>().unwrap_or(log::Level::Info);

    meta.level() <= level
}
```

### `log`

実際にログを出力するメソッドです.
標準出力前提ならば `println!` で出力するだけで問題ありません.
ファイルへ出力する場合はその処理を記述したり, 高速化のために Buffering したりと自由に処理を記述できます.

ドキュメントに記述されているように, `enabled` メソッドが `false` を返す場合であってもこのメソッドは呼び出される可能性があります.
正しく制御するにはこのメソッド内部で `enabled` かどうかを判定する必要があります.

> Note that enabled is not necessarily called before this method. Implementations of log should perform all necessary filtering internally.

```rust
fn log(&self, record: &log::Record) {
    if !self.enabled(record.metadata()) {
        // 出力が有効となる対象でないならば, 何もしない
        return;
    }

    println!("{:>5}: {}", record.level(), record.args());
}
```

このメソッドではエラーの処理を行うことはできません.
エラーが発生した場合には, 標準エラー等にその情報を出力するなど, 代替出力先を事前に検討しておく必要があります.
代替出力も不能となると, エラーを無視するか, panic するかのどちらかになります.

```rust
fn log(&self, record: &log::Record) {
    use std::io::Write;
    use std::fs::OpenOptions;

    if !self.enabled(record.metadata()) {
        return;
    }

    // ログ出力先のファイルを開いて書き込む
    if let Ok(mut file) = OpenOptions::new()
        .create(true)
        .append(true)
        .open("log.txt")
    {
        writeln!(file, "{:>5}: {}", record.level(), record.args()).unwrap_or_else(|e| {
            eprintln!("Error: {}", e);
        });
    } else {
        eprintln!("Error: Failed to open log file");
    }
}
```

### `flush`

出力を Buffering している場合に, Buffer を flush するメソッドです.
例えば `log` メソッドにて `print!` や `write!` を利用している場合は, このメソッド内で `stdout` の `flush` が必要です.

```rust
fn flush(&self) {
    use std::io::Write;

    std::io::stdout().flush().unwrap_or_else(|e| {
        eprintln!("Error: {}", e);
    });
}
```

随時出力を `flush` する `println!` 等のみを `log` メソッドで利用する場合, このメソッドは空で問題ありません.

## GCP 向け Logger 実装試行

今回は GCP の Cloud Logging に合わせた, Structured Logging を行う Logger を実装してみます.

GKE や Cloud Run といった環境上では, 標準出力に JSON 形式で特定のログを出力することで, 構造化されたログを Cloud Logging に記録できます.

https://cloud.google.com/logging/docs/agent/logging/configuration?hl=ja#special-fields

```rust
use chrono::Utc;

pub struct StructureLogger {}

impl StructureLogger {
  pub const fn new() -> StructureLogger {
    StructureLogger {}
  }
}

impl log::Log for StructureLogger {
  fn enabled(&self, _: &log::Metadata) -> bool {
    true
  }

  fn log(&self, record: &log::Record) {
    if !self.enabled(record.metadata()) {
      return;
    }

    println!(
      // severityラベルとメッセージ, Timestampを付与
      r#"{{"severity":"{}","message":"{}","time":"{}"}}"#,
      record.level(),
      record.args(),
      Utc::now().to_rfc3339(),
    );
  }

  fn flush(&self) {}
}
```

この Logger を利用することで, Severity 等の載ったログが出力されます.

```json
$ cargo run
{"severity":"INFO","message":"Hello, world!","time":"2023-11-20T03:28:37.457625728+00:00"}
```

### Actix Web と連携させる

更にリクエストを受けた際のログ出力に `httpRequest` の Field を載せるようにしてみます.

Actix Web から提供されている Logger Middleware はメッセージフォーマットの指定に加え, log target の指定ができます.

https://docs.rs/actix-web/4.4.0/actix_web/middleware/struct.Logger.html

```rust
  // middleware の設定例
  Logger::new(
    r#"{"requestMethod":"%{METHOD}xi","requestUrl":"%{URL}xi","status":%s,"userAgent":"%{User-Agent}i","latency":"%Ts","responseSize":"%b"}"#,
  )
  // METHODやURLはデフォルトで置換が存在しないので, 置換関数を定義する
  .custom_request_replace("METHOD", |req| req.head().method.to_string())
  .custom_request_replace("URL", |req| req.head().uri.to_string())
  // ターゲットを`request_log`にする
  .log_target("request_log")
```

`request_log` がターゲットの場合のみ `httpRequest` フィールドを埋めるよう `log` 関数を変更します.

```rust
  fn log(&self, record: &log::Record) {
    if !self.enabled(record.metadata()) {
      return;
    }

    if record.metadata().target() == "request_log" {
      println!(
        r#"{{"severity":"{}","httpRequest":{},"time":"{}"}}"#,
        record.level(),
        record.args(),
        Utc::now().to_rfc3339(),
      );
    } else {
      println!(
        r#"{{"severity":"{}","message":"{}","time":"{}"}}"#,
        record.level(),
        record.args(),
        Utc::now().to_rfc3339(),
      );
    }
  }
```

これで, リクエストログには `httpRequest` フィールドが載り, 他のメッセージ出力では`message`のみとログが使い分けられるようになりました.

```json
{"severity":"INFO","message":"starting 8 workers","time":"2023-11-20T03:28:37.457625728+00:00"}
{"severity":"INFO","message":"Tokio runtime found; starting in existing Tokio runtime","time":"2023-11-20T03:28:37.457667135+00:00"}
{"severity":"INFO","httpRequest":{"requestMethod":"GET","requestUrl":"/health","status":200,"userAgent":"curl/7.81.0","latency":"0.000081s","responseSize":"0"},"time":"2023-11-20T03:40:46.831461102+00:00"}
```

これを GCP 上で確認すると, それは見事にいい感じに構造化されたログとして確認できます.

### Error Reporting

Rust 1.65 から Backtrace を取得できるようになりましたし, エラー出力時にその Trace を含めてみることもできます.
しかし現状 GCP の Error Reporting は Rust の BackTrace は解析してくれないようですね...

まぁ ErrorReport 自体は生成されるので良しとしましょう.

https://cloud.google.com/error-reporting/docs/formatting-error-messages?hl=ja#log-error

```rust
    if record.metadata().target() == "request_log" {
      // 上記略...
    } else if record.metadata().level() <= log::Level::Error {
      let backtrace = std::backtrace::Backtrace::capture();

      if backtrace.status() == std::backtrace::BacktraceStatus::Captured {
        println!(
          r#"{{"severity":"{}","message":"{}","time":"{}","stack_trace":"{}","@type": "type.googleapis.com/google.devtools.clouderrorreporting.v1beta1.ReportedErrorEvent"}}"#,
          record.level(),
          record.args(),
          Utc::now().to_rfc3339(),
          backtrace.to_string().escape_debug(),
        );
      }
    } else {
      // 上記略...
    }
```

### Logger の分離

せっかく`enabled` などのメソッドがあるにも関わらず, 出力可否の判定を 1 箇所にまとめてしまうと, 可読性が低下してしまいますね.
ということで「複数の Logger を取りまとめる Logger」も実装してみます.

```rust
pub struct JoinLogger {
  // 特定のログに対応する Loggerたち
  loggers: Vec<Box<dyn log::Log>>,
  // 特別対応しない普遍的 Logger
  fallback: Option<Box<dyn log::Log>>,
}

impl JoinLogger {
  pub fn new() -> JoinLogger {
    JoinLogger {
      loggers: Vec::new(),
      fallback: None,
    }
  }

  pub fn add_logger(&mut self, logger: Box<dyn log::Log>) {
    self.loggers.push(logger);
  }

  pub fn set_fallback(&mut self, fallback: Box<dyn log::Log>) {
    self.fallback = Some(fallback);
  }
}

impl log::Log for JoinLogger {
  fn enabled(&self, _: &log::Metadata) -> bool {
    true
  }

  fn log(&self, record: &log::Record) {
    let mut logged = false;

    // 特定のメタデータで有効な Logger があれば, そのログ出力を実行する
    for logger in &self.loggers {
      if logger.enabled(record.metadata()) {
        logger.log(record);
        logged = true;
      }
    }

    // なければ fallback にログ出力を実行させる
    if !logged {
      if let Some(fallback) = &self.fallback {
        fallback.log(record);
      }
    }
  }

  fn flush(&self) {
    for logger in &self.loggers {
      logger.flush();
    }
    if let Some(logger) = &self.fallback {
      logger.flush();
    }
  }
}
```

これで Application, Request, Error それぞれの Logger を定義した上でこれに登録することで, 他の Logger とは干渉せずにログを出力できます.

```rust
/// アプリケーション内のどこからでも利用する Logger
pub struct ApplicationLogger {}

impl ApplicationLogger {
  pub fn new() -> ApplicationLogger {
    ApplicationLogger {}
  }
}

impl log::Log for ApplicationLogger {
  fn enabled(&self, _: &log::Metadata) -> bool {
    true
  }

  fn log(&self, record: &log::Record) {
    if !self.enabled(record.metadata()) {
      return;
    }

    println!(
      r#"{{"severity":"{}","message":"{}","time":"{}"}}"#,
      record.level(),
      record.args(),
      Utc::now().to_rfc3339(),
    );
  }

  fn flush(&self) {}
}
```

```rust
/// エラー発生時のみ利用する Logger
pub struct ErrorLogger {}

impl ErrorLogger {
  pub fn new() -> ErrorLogger {
    ErrorLogger {}
  }
}

impl log::Log for ErrorLogger {
  fn enabled(&self, meta: &log::Metadata) -> bool {
    let backtrace = std::backtrace::Backtrace::capture();
    let backtrace_enabled = backtrace.status() == std::backtrace::BacktraceStatus::Captured;
    let severity_error_or_over = meta.level() <= log::Level::Error;

    backtrace_enabled && severity_error_or_over
  }

  fn log(&self, record: &log::Record) {
    if !self.enabled(record.metadata()) {
      return;
    };

    let backtrace = std::backtrace::Backtrace::capture();
    println!(
      r#"{{"severity":"{}","message":"{}","time":"{}","stack_trace":"{}","@type": "type.googleapis.com/google.devtools.clouderrorreporting.v1beta1.ReportedErrorEvent"}}"#,
      record.level(),
      record.args(),
      Utc::now().to_rfc3339(),
      backtrace.to_string().escape_debug(),
    );
  }

  fn flush(&self) {}
}
```

```rust
/// Actixのリクエストログ用のLogger
pub struct RequestLogger {}

impl RequestLogger {
  pub fn new() -> RequestLogger {
    RequestLogger {}
  }
}

impl log::Log for RequestLogger {
  fn enabled(&self, meta: &log::Metadata) -> bool {
    meta.target() == "request_log"
  }

  fn log(&self, record: &log::Record) {
    if !self.enabled(record.metadata()) {
      return;
    }

    println!(
      r#"{{"severity":"{}","httpRequest":{},"time":"{}"}}"#,
      record.level(),
      record.args(),
      Utc::now().to_rfc3339(),
    );
  }

  fn flush(&self) {}
}
```

```rust
fn logger_init() {
    let mut logger = JoinLogger::new();
    // Request Log, Error Logは専用Loggerに, それ以外はApplication Loggerに流す
    logger.add_logger(Box::new(RequestLogger::new()));
    logger.add_logger(Box::new(ErrorLogger::new()));
    logger.set_fallback(Box::new(ApplicationLogger::new()));

    log::set_boxed_logger(Box::new(logger)).expect("failed to initialize");
    log::set_max_level(log::LevelFilter::Trace);
}
```

ボイラープレート的な部分が増えてはしまいますが, どのログがいつ出力されるのか, どのようなログが出力されるのかを明確にできました.

## まとめ

Rust におけるロギングについて, その基本的な使い方と実装方法を解説しました.
`env_logger`や`fern`, あるいは `trace` といったライブラリを使った方が良いケースは多々ありますが, 自身でカスタム Logger を作るという選択肢もあります.
「しっくり来るものがない!!」「機能過剰!!こんなに要らない!!」となった場合は, ぜひ自作を試してみてください.

[^1]: [Log in log - Rust](https://docs.rs/log/0.4.20/log/trait.Log.html)
