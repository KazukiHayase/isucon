# echo に pprof を導入したときのメモ

## アプリケーションに組み込み

下記記事を参考に

https://zenn.dev/muroon/articles/adf577f563c806
https://zenn.dev/yuucu/scraps/ccaea79d6e90af

これだけではダメ

```go
import(
  _ "net/http/pprof"
)
```

これでもダメ。echo の場合だとハンドラは `echo.HandlerFunc` でないといけないため

```go
import (
    "github.com/gorilla/mux"
    "net"
    "net/http"
    "net/http/pprof" // 追加(importしているだけ)
)

func main() {
    r := mux.NewRouter()
    r.HandleFunc("/debug/pprof/", pprof.Index) // 追加
    r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline) // 追加
    r.HandleFunc("/debug/pprof/profile", pprof.Profile) // 追加
    r.HandleFunc("/debug/pprof/symbol", pprof.Symbol) // 追加
    r.HandleFunc("/debug/pprof/heap", pprof.Handler("heap").ServeHTTP) // 追加
    r.HandleFunc("/", hello())
    lin, err := net.Listen("tcp", ":8080")
    if err != nil {
        panic(err)
    }
    defer lin.Close()
    s := new(http.Server)
    s.Handler = r
    s.Serve(lin)
```

[echo 用にいい感じにしてくれる (公式製？) ライブラリ](https://github.com/labstack/echo-contrib)があるようなので、それを使う

```go
package main

import (
	"net/http"

	"github.com/labstack/echo/v4"
	"github.com/labstack/echo-contrib/pprof"

)

func main() {
	e := echo.New()
	pprof.Register(e)
    ......
	e.Logger.Fatal(e.Start(":1323"))
}
```

## ローカルで実行

webapp を立て、bench を回している最中に下記を実行

```sh
 $ go tool pprof -http=":22222" http://localhost:7002/debug/pprof/profile

Fetching profile over HTTP from http://localhost:7002/debug/pprof/profile
Saved profile in /Users/kmtym1998/pprof/pprof.isucholar.samples.cpu.001.pb.gz
Serving web UI on http://localhost:22222
Failed to execute dot. Is Graphviz installed?
exec: "dot": executable file not found in $PATH
Failed to execute dot. Is Graphviz installed?
exec: "dot": executable file not found in $PATH
```

dot がないと言われた

https://qiita.com/chimayu/items/887709604044d767739e を参考に

```
brew install graphviz
```

できた。ブラウザで開いて Flame Graph (new) を見ればいい感じに可視化が可能

https://github.com/KazukiHayase/isucon/assets/54347899/5b554e55-a40f-4800-b800-98d05ac4d386

本番は向き先を EC2 のグローバル IP に向ければいいのかな？

## EC2 で実行

### pprof を実行する

Echo 用に pprof を入れて・・・ (上記参照)

graphviz を入れて・・・

```sh
$ sudo apt-get update
$ sudo apt install -y graphviz
```

profile の取得をする

```sh
# cpu
$ go tool pprof -seconds="60" -http=":22222" http://localhost:7000/debug/pprof/profile

# memory
$ go tool pprof -seconds="60" -http=":22222" http://localhost:7000/debug/pprof/heap
```

seconds には bench の回っている実行時間よりちょい長めを指定する。これを実行してから bench を回し始めるので。

### pprof の結果を見る

Mac から EC2 のデバッグエンドポイントを叩くと、よくわからんエラーが出て失敗する

```sh
# connection が確立できない
$ go tool pprof -http=":22222" http://ec2-13-115-245-199.ap-northeast-1.compute.amazonaws.com/debug/pprof/profile
Fetching profile over HTTP from http://ec2-13-115-245-199.ap-northeast-1.compute.amazonaws.com/debug/pprof/profile
http://ec2-13-115-245-199.ap-northeast-1.compute.amazonaws.com/debug/pprof/profile: Get "http://ec2-13-115-245-199.ap-northeast-1.compute.amazonaws.com/debug/pprof/profile": read tcp 10.100.65.217:55283->13.115.245.199:80: read: connection reset by peer

# EOF が返る
$ go tool pprof -http=":22222" http://ec2-13-115-245-199.ap-northeast-1.compute.amazonaws.com/debug/pprof/profile
Fetching profile over HTTP from http://ec2-13-115-245-199.ap-northeast-1.compute.amazonaws.com/debug/pprof/profile
http://ec2-13-115-245-199.ap-northeast-1.compute.amazonaws.com/debug/pprof/profile: Get "http://ec2-13-115-245-199.ap-northeast-1.compute.amazonaws.com/debug/pprof/profile": EOF

# https だと証明書関係(?)で失敗する
$ go tool pprof -http=":22222" https://ec2-13-115-245-199.ap-northeast-1.compute.amazonaws.com/debug/pprof/profile
Fetching profile over HTTP from https://ec2-13-115-245-199.ap-northeast-1.compute.amazonaws.com/debug/pprof/profile
https://ec2-13-115-245-199.ap-northeast-1.compute.amazonaws.com/debug/pprof/profile: Get "https://ec2-13-115-245-199.ap-northeast-1.compute.amazonaws.com/debug/pprof/profile": tls: failed to verify certificate: x509: "*.t.isucon.dev" certificate is not standards compliant
```

SSH している EC2 からポートフォワーディングする方針に

SG の設定を変える、22222 番ポートを開けてもらう。

ローカル Mac で下記を実行する

```sh
# localhost:9999 → ec2-13-115-245-199.ap-northeast-1.compute.amazonaws.com:22222 に転送できるようになる
$ ssh -i ~/.ssh/isucon -L 9999:localhost:22222 ubuntu@ec2-13-115-245-199.ap-northeast-1.compute.amazonaws.com
```
