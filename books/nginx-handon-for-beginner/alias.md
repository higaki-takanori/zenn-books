---
title: "リクエストに応じて参照するディレクトリを変更"
---

# 目的

- aliasを使用し、リクエストに応じて参照するディレクトリを変更する

# 画像配信

この章では、リクエストによって参照するディレクトリを変更する設定を行う。

画像のリクエストは参照するディレクトリを変更する。

## ディレクトリの作成

nginx配下にimgディレクトリを作成

```
$ mkdir img
```

```
.
├── docker-compose.yml
└── nginx
    ├── conf.d
    │   └── static.conf
    ├── html
    │   ├── 404.html
    │   ├── 50x.html
    │   ├── docs
    │   │   └── sub.html
    │   └── index.html
    └── img <----これを作成
```

## 画像のダウンロード

imgディレクトリ配下に何かしらの画像を用意してください。

一応、画像をダウンロードするコマンドを参考に載せておきます。

```
$ curl https://www.python.org/static/apple-touch-icon-144x144-precomposed.png > test.png
```

## 設定ファイルの変更

```diff:static.conf
server {
  listen 8000;

  server_name www.example.com;

  access_log /var/log/nginx/www.example.com_access.log;
  error_log /var/log/nginx/www.example.com_error.log;

  root /var/www/html;

  error_page 404 /404.html;
  error_page 500 502 503 504 /50x.html;

+  location /images/ {
+    alias /var/www/img/;
+  }
}
```

これにより、`/image/`のリクエストは`/var/www/img`配下を参照するようになる。

:::message
`alias`の代わりに`root`を使用した場合

```
root /var/www/img/;
```

とすると、`localhost:8000/images/test.png`のリクエスト要求のファイルが

```
/var/www/img/images/test.png
```

となりエラーとなる。
:::

## 動作確認

nginxの設定を適用させるコマンドを入力する。順次適用されていく。

```
(nginxコンテナ)# nginx -s reload
```

ブラウザで`localhost:8000/images/test.png`にアクセスすると、

ダウンロードした画像が表示される。

# この章の完了後のnginxの設定

:::details nginxの設定
```
# configuration file /etc/nginx/nginx.conf:

user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}

# configuration file /etc/nginx/mime.types:

types {
    text/html                                        html htm shtml;
...省略
    video/x-msvideo                                  avi;
}

# configuration file /etc/nginx/conf.d/static.conf:
server {
  listen 8000;

  server_name www.example.com;

  access_log /var/log/nginx/www.example.com_access.log main;
  error_log /var/log/nginx/www.example.com_error.log;

  root /var/www/html;

  error_page 404 /404.html;
  error_page 500 502 503 504 /50x.html;

  location /images/ {
    alias /var/www/img/;
  }
}
```
:::


# この章の完了後のディレクトリ構造

```
.
├── docker-compose.yml
└── nginx
    ├── conf.d
    │   └── static.conf
    ├── html
    │   ├── 404.html
    │   ├── 50x.html
    │   ├── docs
    │   │   └── sub.html
    │   └── index.html
    └── img
        └── test.png
```
