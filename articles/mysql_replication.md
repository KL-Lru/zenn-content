---
title: "動かして学ぶ MySQL Replication"
emoji: "🐬"
type: "tech"
topics: ["MySQL", "Replication"]
published: true
---

## はじめに

レプリケーションとは, 特定の MySQL データベースサーバから, 別の MySQL データベースサーバへデータを継続的に複製しつづけることのできる機能です.
本記事では Master - Slave 型 (単一の Master, 複数の Slave を持つ構成) のレプリケーションについて試して挙動を見てみます.

## ソースコード

本記事に登場する設定やスクリプトは, 全てこちらのリポジトリ上に展開されています.

https://github.com/KL-Lru/mysql-replication-example

当記事内に記載されている作業と同等の内容をすべて `Docker`と`Makefile`を利用して再現試行できます.
細かな手順は README を参照ください.

## レプリケーションの形式

現世でのレプリケーションでは次のような選択肢があります. [^1]

- binlog の位置を直接指定する形でレプリケーションを開始する
- GTID を利用してレプリケーションを開始する

どちらの場合でも基本的には Master で記録された binlog を参照し, Slave 上で relaylog を生成, それを元に Master 上のイベントを Slave で再現します.

他にもサードパーティのツールを利用してレプリケーションを行うことも出来ますが, 本記事では脱線するため割愛します.
(リポジトリには Debezium を利用した例を含めています. そちらの解説はまた後日. )

## binlog の位置ベースのレプリケーション [^2]

古くから存在している方法です.
binlog のどのイベントから再現してよいかを直接指定する方法です.

### 各 MySQL サーバの設定

レプリケーションを行うにあたって, 次の条件を満たす必要があります.

- Master となるサーバで binlog が有効になっている
- 各 MySQL サーバの `server_id`が一意になっている

その他, 安全にレプリケーションを行うための設定としては次のようなものがあります.

| オプション                       | 意図                                                                                                         |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `log_bin`                        | binlog を有効にする. (MySQL 8.0 以降はデフォルト有効) <br /> `my.cnf`上では指定するのは出力されるログの名称. |
| `binlog_format`                  | binlog の形式を指定する. <br /> レプリケーションの安全性が最も高いのは`ROW`                                  |
| `innodb_flush_log_at_trx_commit` | InnoDB ログのファイルへのフラッシュタイミングを指定する. <br />`1`: 各トランザクション毎にフラッシュ         |
| `sync_binlog`                    | バイナリログのディスクへの同期タイミングを指定する. <br /> `1`: 各トランザクション毎にフラッシュ             |
| `sync_relay_log`                 | リレーログのディスクへの同期タイミングを指定する. <br /> `1`: 各イベント毎にフラッシュ                       |

:::message
ファイルへのフラッシュ制御については書込頻度を上げるほど安全性は増しますがパフォーマンスが低下します.
:::

設定の一例としては次のような形になります.

```toml:master.cnf
[mysqld]
# Server ID が一意になるように設定
server_id=1

# binlog を有効にする
log_bin=mysql-bin

# ROW-based で binlog を記録
binlog_format=ROW

# ディスクへの同期を最も安全に設定
innodb_flush_log_at_trx_commit=1
sync_binlog=1
```

```toml:slave.cnf
[mysqld]
# Server ID が一意になるように設定
server_id=2

# Slaveに直接書き込み出来ないようにする
read_only=ON

# ディスクへの同期を最も安全に設定
sync_relay_log=1
```

### レプリケーション処理用のユーザの作成

Slave は Master 上のユーザを利用して Master に接続するため, そのためのユーザが必要になります.
必要となる権限は`REPLICATION SLAVE`のみです.

```sql
-- User for replication
CREATE USER IF NOT EXISTS 'repl'@'%' IDENTIFIED BY 'repl_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
```

### binlog の位置の確認

Master 上で次のクエリを実行し, binlog の位置を確認します.

```sql
SHOW MASTER STATUS;
```

次のような出力が得られ, このうち`File`と`Position`を控えておきます.
これが確認時点での binlog の位置になります.

```sql
--------------
SHOW MASTER STATUS
--------------

*************************** 1. row ***************************
             File: mysql-bin.000003
         Position: 1253
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
```

すでにデータが存在する場合は dump と, それを出力した際の binlog の位置を控えておくことで, dump を流し込んだ後のレプリケーション開始時にその位置から処理を開始できます.
この手段を取る場合には, `FLUSH TABLES WITH READ LOCK;`を実行して書き込みをすべて完了した状態でロックするなどの対策が必要になります.

### レプリケーションの開始

Slave 上で次のクエリを実行し, レプリケーションを開始します.

```sql
-- Setup replication configuration
STOP REPLICA;
RESET REPLICA;

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 'master',
  SOURCE_PORT = 3306,
  -- 最初に作成したMaster上のユーザ
  SOURCE_USER = 'repl',
  SOURCE_PASSWORD = 'repl_password',
  -- 確認したbinlogの位置
  SOURCE_LOG_FILE = 'mysql-bin.000003',
  SOURCE_LOG_POS = 1253;

-- Start replication
START REPLICA;
```

この SQL を実行した段階から, Master 上での変更が Slave にも反映されるようになります.

### レプリケーションの確認

Slave 上で次のクエリを実行し, レプリケーションの状態を確認します.

```sql
SHOW REPLICA STATUS\G
```

```
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: master
                  Source_User: repl
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: mysql-bin.000003
          Read_Source_Log_Pos: 1253
               Relay_Log_File: 23a5eb08c7a1-relay-bin.000002
                Relay_Log_Pos: 326
        Relay_Source_Log_File: mysql-bin.000003
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
          ...
```

Source として Master が指定され, それに対しての接続情報が確認できます.

### 動作検証

Master 上で次のクエリを実行し, データを追加します.

```sql
CREATE TABLE IF NOT EXISTS users (
  id VARCHAR(36) NOT NULL,
  name VARCHAR(255) NOT NULL,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO users (id, name) VALUES (UUID(), 'John'), (UUID(), 'Jane'), (UUID(), 'Bob'), (UUID(), 'Alice'), (UUID(), 'Eve');

UPDATE users SET name = CONCAT(name, ' Doe') WHERE name = 'John' OR name = 'Jane';
```

Slave 上で次のクエリを実行し, データが反映されていることを確認します.

```sql
SELECT * FROM users;
```

```
id      name
aa1779e7-cce7-11ee-bda6-0242ac180003    John Doe
aa177c38-cce7-11ee-bda6-0242ac180003    Jane Doe
aa177cb5-cce7-11ee-bda6-0242ac180003    Bob
aa177ce4-cce7-11ee-bda6-0242ac180003    Alice
aa177d0f-cce7-11ee-bda6-0242ac180003    Eve
```

Master 上での変更が Slave 上にも反映されていることが確認できます.

## GTID を利用したレプリケーション [^3]

GTID (Global Transaction ID)は MySQL サーバのトランザクションに対して付与される一意な識別子です.
`source_id:transaction_id`という形式で表現され, Master / Slave を含み, 構成上のすべてのサーバ, すべてのトランザクションに対して一意になります.
この GTID は, どのサーバでどのトランザクションがまだ適用されていないかなどを識別する材料となります.

GTID が存在することで, binlog の位置の直接指定ベースのレプリケーションと比較して, レプリケーションの開始や停止, 途中での切り替えなどがはるかに容易になります.

### 各 MySQL サーバの設定

GTID を利用するためには, 次の条件を満たす必要があります.

- Master となるサーバで binlog が有効になっている
- 各 MySQL サーバの `server_id`が一意になっている
- `gtid_mode`が`ON`になっている
- `enforce_gtid_consistency`が`ON`になっている
  - `gtid_mode`が`ON`の場合, `enforce_gtid_consistency`は`ON`でなければなりません.

他の多くのオプションは, binlog の位置の直接指定ベースのレプリケーションと同様に設定できます.

```toml:master.cnf
[mysqld]
# Server ID が一意になるように設定
server_id=1

# binlog を有効にする
log_bin=mysql-bin

# ROW-based で binlog を記録
binlog_format=ROW

# ディスクへの同期を最も安全に設定
innodb_flush_log_at_trx_commit=1
sync_binlog=1

# GTID を有効にする
gtid_mode=ON
enforce_gtid_consistency=ON
```

```toml:slave.cnf
[mysqld]
# Server ID が一意になるように設定
server_id=2

# Slaveに直接書き込み出来ないようにする
read_only=ON

# ディスクへの同期を最も安全に設定
sync_relay_log=1

# GTID を有効にする
gtid_mode=ON
enforce_gtid_consistency=ON
```

### レプリケーション処理用のユーザの作成

binlog の位置ベースのレプリケーションと同様に, Slave は Master 上のユーザを利用して Master に接続するため, そのためのユーザが必要になります.
権限も全く同一です.

```sql
-- User for replication
CREATE USER IF NOT EXISTS 'repl'@'%' IDENTIFIED BY 'repl_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
```

### レプリケーションの開始

GTID を利用する場合, どのサーバで発生したトランザクションかは自動的に識別可能なため, 位置の特定などの設定は不要です.
次のクエリを実行することで, レプリケーションを開始できます.

```sql
-- Setup replication configuration
STOP REPLICA;
RESET REPLICA;

CHANGE REPLICATION SOURCE TO
  SOURCE_SSL = 1,
  SOURCE_HOST = 'master',
  SOURCE_PORT = 3306,
  -- 最初に作成したMaster上のユーザ
  SOURCE_USER = 'repl',
  SOURCE_PASSWORD = 'repl_password',
  -- GTIDを利用して自動的に位置を特定
  SOURCE_AUTO_POSITION = 1;

-- Start replication
START REPLICA;
```

### レプリケーションの確認

Master / Slave それぞれでステータスを確認してみます.

```sql
--------------
SHOW MASTER STATUS
--------------

*************************** 1. row ***************************
             File: mysql-bin.000003
         Position: 1291
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set: 5d7eae25-cceb-11ee-bf85-0242ac1a0002:1-10

--------------
SHOW REPLICA STATUS
--------------

*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: master
                  Source_User: repl
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: mysql-bin.000003
          Read_Source_Log_Pos: 1291
               Relay_Log_File: 901e5b6a577a-relay-bin.000003
                Relay_Log_Pos: 1507
        Relay_Source_Log_File: mysql-bin.000003
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
        ...
```

自動的に binlog の位置が特定され, それに基づいてレプリケーションが設定されていることが確認できます.

### 動作検証

binlog の位置ベースのレプリケーションと同様のことが起こります. (割愛)

## まとめ

MySQL のレプリケーションについて, その基本的な動作と設定方法について確認し, それを実際に試してみました.
普段クラウドのマネージドサービスのボタンポチーで出来上がるレプリカの裏側で何が起こっているのか, その一端だけでも理解できたような気がします.

Docker のおかげでこういった検証作業も容易に行えるのはとても助かりますね.
時間が出来たら, 複数 Master のレプリケーションなんかも試してみたいところです.

[^1]: [MySQL :: MySQL 8.0 Reference Manual :: レプリケーションの構成](https://dev.mysql.com/doc/refman/8.0/ja/replication-configuration.html)
[^2]: [MySQL :: MySQL 8.0 Reference Manual :: バイナリログファイルの位置ベースのレプリケーション構成](https://dev.mysql.com/doc/refman/8.0/ja/binlog-replication-configuration-overview.html)
[^3]: [MySQL :: MySQL 8.0 Reference Manual :: グローバルトランザクション識別子を使用したレプリケーション](https://dev.mysql.com/doc/refman/8.0/ja/replication-gtids.html)