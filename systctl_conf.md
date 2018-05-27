#study_work 宿題

## sysctl.confのチューニング

### sysctlとは
Linuxカーネルの設定をいじるためのものらしい。
`sysctl`コマンドを使うか、`/etc/sysctl.conf`に設定を書き込んで`sudo /sbin/sysctl -p`コマンドで反映することで設定を変更できる。
あるいは`/proc` 以下の各ディレクトリツリーのファイル内に必要な値を記述する方法もあり、こちらは即時反映されるが、再起動すると設定が元に戻る。


### チューニング
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



## 試す

設定前 ：321.9

設定後 ：323.4
