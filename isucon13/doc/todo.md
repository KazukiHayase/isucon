# インスタンスを立ち上げたらやること

1. Makefile作成
2. ssh key登録
3. `make init`
4. 必要なファイル群をadd+push
5. 各インスタンスでnginxのシンボリック作成 + push

```bash
cp /etc/nginx/nginx.conf /home/isucon/<app_name>/webapp/nginx/nginx_1.conf
sudo ln -sb /home/isucon/<app_name>/webapp/nginx/nginx_1.conf /etc/nginx/nginx.conf
```
5. nginx のログ出力設定

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

access_log  /tmp/access.log  ltsv;
```

6. デプロイして初回ベンチ実行
