# study_work_1

## 環境構築
稲垣がISUCON5の環境を作ってくれたので、それを入れる。

```
git clone https://github.com/gky360/vagrant-isucon.git
cd vagrant-isucon/isucon5-qualifier
vagrant up image
```

`vagrant up`に数時間かかる。

http://192.168.33.10 が見られたらここまではok。

続いてベンチマーク用の環境を構築

```
vagrant up bench
vagrant ssh bench
sudo su -l isucon
```

上で立てた競技用ウェブアプリに対してベンチマークを走らせる
```
./bench.sh 192.168.33.10
```

## ユーザ環境構築
便利なツールを入れる。

```
sudo apt-get install -y zsh vim tmux htop dstat
```
`zsh`は便利なシェル
`vim`はエディタ
`tmux`はターミナルの拡張的な
`htop`, `dstat`はCPUの使用率とか見れる。

## goに変える
デフォルトではrubyが走っている。
goに変更する。

```
# isucon ユーザに変える <- 毎回やる
sudo su -l isucon

# systemctl スクリプトの確認
ls -l /etc/systemd/system/

# 80 と 8080 で何か動いている
sudo lsof -i :8080

# 実はこの vagrant はデフォルトで ruby 版の実装が動いている
systemctl status isuxi.ruby.service

# ruby 版webapp停止
systemctl stop isuxi.ruby.service
systemctl disable isuxi.ruby.service
sudo lsof -i :8080

# golang 用環境設定
echo "export PATH=/home/isucon/.local/go/bin:$PATH" >> ~/.bashrc
echo "export GOROOT=/home/isucon/.local/go" >> ~/.bashrc
echo "export GOPATH=/home/isucon/webapp/go" >> ~/.bashrc
exec -l $SHELL

# go build
cd ~/webapp/go
go build app.go

# golang のサーバに切り替え
systemctl start isuxi.go.service
systemctl status isuxi.go.service
sudo lsof -i :8080
```

`lsof -i` でポート指定で動いているプロセスを確認できる。

`systemctl`で動かしたり止めたりできる。
`systemctl stop`で止めて、`systemctl disable`で再起動時に勝手に起動する設定を消す。

## ログを追加する
デフォルトではサービスがログを吐かないので、`app.go`に修正を加えてログを吐くようにする。

`~/webapp/go/app.go`に以下の関数を追加。
追加するのはどこでもいいが、例えばmain()関数手前などがわかりやすい。

```
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Do stuff here
        log.Println(r.RequestURI)
        // Call the next handler, which can be another middleware in the chain, or the final handler.
        next.ServeHTTP(w, r)
    })
}
```

次に、main()関数の`r := mux.NewRouter()`以降に以下の二行を追加。

```
r.Use(loggingMiddleware)
log.Println("Started server")
```

これで、でログを吐くようになる。

また、nginxのログは以下のコマンドで見れる。
```
sudo tail -F /var/log/nginx/access.log
```
