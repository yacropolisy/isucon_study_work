---
title: ISUCON5 予選 part3
tags: isucon MySQL nginx
author: gky360
slide: false
---
研究室有志によるISUCON勉強会 ISUCON部 の資料です。

## 概要

今回は、scoreはあまり気にせず、インフラ側のチューニングの手数を増やすことが目的。以下のような事をやってみる。

- mysql
    - /etc/my.cnf いじる
    - indexを張る
- /etc/sysctl.conf でネットワーク設定いじる
- nginx.conf
    - keepalive, スレッド数, ファイルディスクリプタのキャッシュ, TCP_CORKによるデータ書き出しタイミングの変更、など


## スコア遷移

| | scores |
|---|---|
| 修正前 | 276.8, 332.4, 405.0 |
| my.cnf 修正 | 343.5, 377.8, 391.4 (?) |
| mysql で unix socket を使う | 368.3, 377.8, 409.2 (?) |
| nginx.conf 修正 | 390.5, 454.6, 463.9 (?) |
| nginx で unix socket を使う | 471.9, 479.3, 507.8 |
| mysql に index 張る | 572.7, 585.1, 651.9 |
| sysctl.conf を修正 | 653.3, 654.9, 665.0 |

## mysql のチューニング

### index を追加

今回は `mysqldumpslow` で合計実行時間が一番長かった以下のクエリに着目する。

```
Count: 150  Time=0.74s (110s)  Lock=0.00s (0s)  Rows=201.9 (30288), root[root]@localhost
# 32.3s user time, 50ms system time, 28.73M rss, 85.54M vsz                              │  SELECT * FROM relations WHERE one = N OR another = N ORDER BY created_at DESC
```


`relations` テーブルの index の情報を見てみる。

```
mysql> show index from relations;
+-----------+------------+--------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table     | Non_unique | Key_name           | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-----------+------------+--------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| relations |          0 | PRIMARY            |            1 | id          | A         |      483182 |     NULL | NULL   |      | BTREE      |         |               |
| relations |          0 | friendship         |            1 | one         | A         |       10280 |     NULL | NULL   |      | BTREE      |         |               |
| relations |          0 | friendship         |            2 | another     | A         |      483182 |     NULL | NULL   |      | BTREE      |         |               |
+-----------+------------+--------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
```

id が primary_key に, (one, another) が複合ユニークkeyになっている模様。

explain でクエリの情報を見てみると、keyを何も使ってくれていないことがわかる。

```
mysql> explain SELECT * FROM relations WHERE one = 10 OR another = 10 ORDER BY created_at DESC;
+----+-------------+-----------+------+---------------+------+---------+------+--------+-----------------------------+
| id | select_type | table     | type | possible_keys | key  | key_len | ref  | rows   | Extra                       |
+----+-------------+-----------+------+---------------+------+---------+------+--------+-----------------------------+
|  1 | SIMPLE      | relations | ALL  | friendship    | NULL | NULL    | NULL | 483182 | Using where; Using filesort |
+----+-------------+-----------+------+---------------+------+---------+------+--------+-----------------------------+
```

ということで、この select クエリ時に働いてくれるような index を張る。今回は、 (one, created_at) と (another, created_at) の2つの複合indexを張る。

```
mysql> ALTER TABLE relations ADD INDEX one_created_at(one, created_at);
Query OK, 0 rows affected (4.92 sec)
Records: 0  Duplicates: 0  Warnings: 0
mysql> ALTER TABLE relations ADD INDEX another_created_at(another, created_at);
Query OK, 0 rows affected (5.34 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

```
mysql> show index from relations;
+-----------+------------+--------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table     | Non_unique | Key_name           | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-----------+------------+--------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| relations |          0 | PRIMARY            |            1 | id          | A         |      483182 |     NULL | NULL   |      | BTREE      |         |               |
| relations |          0 | friendship         |            1 | one         | A         |       10280 |     NULL | NULL   |      | BTREE      |         |               |
| relations |          0 | friendship         |            2 | another     | A         |      483182 |     NULL | NULL   |      | BTREE      |         |               |
| relations |          1 | one_created_at     |            1 | one         | A         |       10066 |     NULL | NULL   |      | BTREE      |         |               |
| relations |          1 | one_created_at     |            2 | created_at  | A         |      483182 |     NULL | NULL   |      | BTREE      |         |               |
| relations |          1 | another_created_at |            1 | another     | A         |       10066 |     NULL | NULL   |      | BTREE      |         |               |
| relations |          1 | another_created_at |            2 | created_at  | A         |      483182 |     NULL | NULL   |      | BTREE      |         |               |
+-----------+------------+--------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
```

再び explain してみる。

```
mysql> explain SELECT * FROM relations WHERE one = 10 OR another = 10 ORDER BY created_at DESC;
+----+-------------+-----------+-------------+----------------------------------------------+-------------------------------+---------+------+------+------------------------------------------------------------------------------+
| id | select_type | table     | type        | possible_keys                                | key                           | key_len | ref  | rows | Extra                                                                        |
+----+-------------+-----------+-------------+----------------------------------------------+-------------------------------+---------+------+------+------------------------------------------------------------------------------+
|  1 | SIMPLE      | relations | index_merge | friendship,one_created_at,another_created_at | friendship,another_created_at | 4,4     | NULL |  208 | Using sort_union(friendship,another_created_at); Using where; Using filesort |
+----+-------------+-----------+-------------+----------------------------------------------+-------------------------------+---------+------+------+------------------------------------------------------------------------------+
```

結局、 (one, created_at) の方のindexは使ってくれておらず、 (one, another) と (another, created_at) の2つのindexを使ってくれている。注目すべきは `rows` のところ。 `rows` はテーブルから読み出す行数の見積もりになっている。最初のexplainの結果と比較すると、483182 -> 208 に激減していることがわかる。


### my.cnf の修正

参考:
- [[ISUCON用メモ] MySQL - Qiita](https://qiita.com/stomk/items/6265e9fdfdfb4f104a7e)
- [ISUCON Cheat Sheet · GitHub](https://gist.github.com/south37/d4a5a8158f49e067237c17d13ecab12a)

```
innodb_buffer_pool_size=1G
innodb_flush_log_at_trx_commit=0
innodb_flush_method=O_DIRECT
innodb_support_xa=OFF
max_connections=10000
```

#### [innodb_buffer_pool_size=1G](https://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_buffer_pool_size)

InnoDB のキャッシュのサイズを設定。マシンの物理メモリサイズの 80% 程度が目安。

#### [innodb_flush_log_at_trx_commit=0](https://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_flush_log_at_trx_commit)

ログ書き込みのタイミングを制御することで、完璧なACIDではなくなるがパフォーマンスがよくなる。

| 値 | log buffer -> log file | log file -> disk |
|---|---|---|
| 1 (default) | commit毎 | commit毎 |
| 2 | commit毎 | 約1秒毎 |
| 0 | 約1秒毎 | 約1秒毎 |

例えば 0 に設定した場合、diskへの保存が commit毎から約1秒毎になる。log fileへのアクセスが減るが、mysqldが死んだときに約1秒分のtransactionのデータを失う。

#### [innodb_flush_method=O_DIRECT](https://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_flush_method)

OSでのファイルキャッシュを利用しないように設定。

LinuxはデフォルトでディスクI/Oをキャッシュしてくれている。しかし、RDBMSの場合はキャッシュを自前で管理するので、OSのキャッシュはオフにする。
参考: http://tech.nikkeibp.co.jp/it/article/Keyword/20070207/261244/

#### [innodb_support_xa=OFF](https://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_support_xa)

logに書き込む順番を保証しなくする代わりにdisk flushを減らせる。

#### [max_connections=10000](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_max_connections)

client からの同時接続数を増やす。

ちなみに、現在の `max_connections` の値は `show variables like 'max_connections';` で確認できる。

### go -> mysql の接続を tcp socket から unix domain socket に変更

#### socket 場所の確認

```sh
$ mysql_config --socket
/var/run/mysqld/mysqld.sock
```

#### my.cnf の修正

```
socket=/var/run/mysqld/mysqld.sock
```

#### app.go の修正

app.go を修正。詳細は [こちら](https://github.com/gky360/isucon-5-qual-app/commit/9dec8272705544a134c03e312cb965cefd0711fb)

```go
db, err = sql.Open("mysql", user+":"+password+"@unix(/var/run/mysqld/mysqld.sock)/"+dbname+"?loc=Local&parseTime=true")
```

`go build app.go` を忘れずに。


## nginx のチューニング

### nginx.conf の修正

参考:
- [Module ngx_http_core_module](http://nginx.org/en/docs/http/ngx_http_core_module.html#etag)
- [ISUCON4 予選でアプリケーションを変更せずに予選通過ラインを突破するの術 - Hateburo: kazeburo hatenablog](https://kazeburo.hatenablog.com/entry/2014/10/14/170129)
- [ISUCON Cheat Sheet · GitHub](https://gist.github.com/south37/d4a5a8158f49e067237c17d13ecab12a)

[最終的な編集箇所はこちら](https://github.com/gky360/isucon5-qual-etc/commit/b20b5fc5f445600db213c374b025bcf901f71118)

#### sendfile

onにすると、カーネル空間内でファイルの読み込みと送信が完了するようになる。

#### tcp_nopush

onにすると、レスポンスヘッダとファイルの内容をまとめて送るようになり、少ないパケット数で効率良く送れるようになる。sendfileがonのときのみ有効。

#### keepalive

- 参考: [Nginx の keep-alive の設定と検証 - How old are you?](http://www.nari64.com/?p=579)

一定条件を満たす複数リクエストを1 connectionにまとめる。

普通 HTTP webページ は1回のアクセスで複数のリクエストが発生する（画像, css, js, ...）ので、keepaliveを設定するとconnection確立のオーバーヘッドを減らせる。

#### tcp_nodelay

onにするとパケットサイズが小さくても遅延させずに送信する。これにより、1リクエストあたり最大0.2秒節約できる。

通常は、あるタイミングで送信したいデータサイズが小さい場合は最大0.2秒のあいだ次のデータを待ってまとめてパケットにする（Nagle's algorithm）という制御が行われる。

#### mime type の include

MIMEタイプと拡張子の関連付けをインポートする。

#### open_file_cache

一度開いたファイルディスクリプタを再利用できる。

### nginx -> go の接続を tcp socket から unix domain socket に変更

参考:
[Golang で書いた Web アプリケーションを UNIX ドメインソケットで公開 - at kaneshin](http://blog.kaneshin.co/entry/2016/05/29/020302)

#### unix側の設定

ソケットファイルの置き場を作成する。 `/etc/tmpfiles.d/isuxi.conf` を作成して設定を書く。

```sh
$ cat /etc/tmpfiles.d/isuxi.conf
d /var/run/isuxi 0755 root root -
```

その後、設定ファイルを登録 & 反映

```sh
$ sudo systemd-tmpfiles --create /etc/tmpfiles.d/isuxi.conf
$ sudo systemctl daemon-reload
```

#### app.go の修正

- unix domain socket を listen する
- unix domain socket を close する

部分を自前で実装します。参考:
[Golang で書いた Web アプリケーションを UNIX ドメインソケットで公開 - at kaneshin](http://blog.kaneshin.co/entry/2016/05/29/020302)

詳細は [こちら](https://github.com/gky360/isucon-5-qual-app/commit/987e54a3a21e5f76bebebcad9acc4294aff0d1bb)

#### nginx.conf の修正

```
upstream app {
  server unix:/var/run/isuxi/go.sock;
}
```

詳細は [こちら](https://github.com/gky360/isucon5-qual-etc/commit/4b070c0ae32a3dbc6394e87dff271b82fecd5fd7)

### sysctl.conf の修正


#### sysctlとは
Linuxカーネルの設定をいじるためのものらしい。
`sysctl`コマンドを使うか、`/etc/sysctl.conf`に設定を書き込んで`sudo /sbin/sysctl -p`コマンドで反映することで設定を変更できる。
あるいは`/proc` 以下の各ディレクトリツリーのファイル内に必要な値を記述する方法もあり、こちらは即時反映されるが、再起動すると設定が元に戻る。


#### チューニング
参考：https://kazeburo.hatenablog.com/entry/2014/10/14/170129

>ベンチマークツールのhttp keepaliveが無効なので、sysctrl.conf でephemeral portを拡大したり、TIME_WAITが早く回収されるように変更する

```
net.ipv4.tcp_max_tw_buckets = 2000000
net.ipv4.ip_local_port_range = 10000 65000
net.core.somaxconn = 32768
net.core.netdev_max_backlog = 8192
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 10
```

>ついでに、backlogも増やしている。

全然わからなかったので調べた。
https://qiita.com/sion_cojp/items/c02b5b5586b48eaaa469

|パラメーター|内容|default値|
|:---|:---|:---|
|net.ipv4.tcp_max_tw_buckets|システムが同時に保持するtime-waitソケットの最大数。<br>DoS攻撃を防げるため、低くするのはやめたほうが良い。|動的|
|net.ipv4.ip_local_port_range|TCP/IPの送信時に使用するポートの範囲<br>可能なら 1024-65535が良いが、iptablesやAWSだとNetworkACLとかで<br>同様に必要に応じて開放しておかないとパケットが途中で止まってハマるから注意。<br>$ cat /proc/sys/net/ipv4/ip_local_port_rangeで確認できる|32768	61000|
|net.core.somaxconn|一度に受け入れられるTCPセッションのキューの数。<br>TCPのセッション数をbacklogで管理し、それを超えたものはキューに格納される。<br>このキューの数をここで設定する|128|
|net.core.netdev_max_backlog|パケット受信時にキューに繋ぐことができるパケットの最大数|1000|
|net.ipv4.tcp_tw_reuse|自分からの接続を使い回す。tcp_tw_recycleは相手からの接続を使い回す。|0|
|net.ipv4.tcp_fin_timeout|タイムアウトはできる限り短いほうがいい|60|
