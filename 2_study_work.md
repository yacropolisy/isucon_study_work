#study_work_2

## alpなどインストール
```
vagrant ssh image
sudo su - isucon
```
```
sudo apt-get install -y htop dstat glances
sudo apt-get install -y unzip

# alp
mkdir -p ~/tmp
cd ~/tmp
wget https://github.com/tkuchiki/alp/releases/download/v0.3.1/alp_linux_amd64.zip
unzip alp_linux_amd64.zip
sudo install ./alp /usr/local/bin
alp --help
```

##まずnginx の log format 変更してalpで見れるようにする

`/etc/nginx/nginx.conf`を編集する。

[参考](https://github.com/gky360/isucon5-qual-etc/commit/1b7a1c334b0744fb7561461e530df224b5c02cae)

`http {}`内に以下を追加。

```
  log_format ltsv "time:$time_local"
    "\thost:$remote_addr"
    "\tforwardedfor:$http_x_forwarded_for"
    "\treq:$request"
    "\tstatus:$status"
    "\tmethod:$request_method"
    "\turi:$request_uri"
    "\tsize:$body_bytes_sent"
    "\treferer:$http_referer"
    "\tua:$http_user_agent"
    "\treqtime:$request_time"
    "\tcache:$upstream_http_x_cache"
    "\truntime:$upstream_http_x_runtime"
    "\tapptime:$upstream_response_time"
    "\tvhost:$host";

  access_log /var/log/nginx/access.log ltsv;
```

###確認
```
sudo rm /var/log/nginx/access.log && systemctl reload nginx
systemctl status nginx  
```
### alpでみる
`sudo alp -f /var/log/nginx/access.log`

##静的ファイルの無駄をなくす

静的ファイルは普通はCDNとかで配られるが、今回はgoが配ってる
→nginxでやらせる

`/etc/nginx/nginx.conf`を編集し、`server{}`内に以下に追加。

```
location ~ ^/(img|css|js|favicon.ico) {
   root /home/isucon/webapp/static;
}
```

再起動する
`sudo rm /var/log/nginx/access.log && systemctl reload nginx`

今回のではあんまり変わらん。




### mysqlで遅いログをチェックする

mysql slow logを作る  
`/etc/mysql/my.cnf` を編集して以下を追加

```
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 0
```

リスタートする
```
sudo systemctl restart mysql
sudo systemctl restart isuxi.go.service
```

下記コマンドで重いクエリだけ見れる  

`sudo mysqldumpslow /var/log/mysql/slow.log | less`

下記でpt-query-digestを入れて実行すると解析してくれる

`sudo apt-get install percona-toolkit`

`sudo pt-query-digest --limit 10 /var/log/mysql/slow.log`


##クエリを軽くするテクニック
\*でとってるところを一部のとこだけとるとか
indexついてるところでとってくる

`EXPLAIN SELECT ~~ `
とすると何秒かかったかとかがわかる。
