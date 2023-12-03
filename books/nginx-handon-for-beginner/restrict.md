---
title: "アクセス禁止・制限"
---

# 目的

- アクセスを禁止・制限する

# アクセス禁止

アクセスを禁止する設定を行う。

## ディレクトリの作成

html配下にsecretディレクトリを作成

```
$ mkdir secret
$ cd secret
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
    │   └── secret <---- これを作成
    │   └── index.html
    └── img
        └── test.png
```

## ファイルの作成

`secret`ディレクトリ配下にファイルを作成。

```
$ touch secret-file.html
```

```html:secret-file.htmlの内容
<h1>This is Secret File</h1>
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

  location /images/ {
    alias /var/www/img/;
  }

+  location /secret/ {
+    # エラーページを応答する
+    return 404;
+  }
}
```

これにより、`/secret/`のリクエストは404エラーを返却するようになる。

## 動作確認

nginxの設定を適用させるコマンドを入力する。順次適用されていく。

```
(nginxコンテナ)# nginx -s reload
```

ブラウザで`localhost:8000/secret/secret-file.html`にアクセスすると、

404.htmlの内容が表示される。

# アクセス制限

アクセスを制限する設定を行う。

## ディレクトリ作成

html配下にadminディレクトリを作成

```
$ mkdir admin
$ cd admin
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
    │   ├── admin <---- これを作成
    │   ├── docs
    │   │   └── sub.html
    │   ├── index.html
    │   └── secret
    │       └── secret-file.html
    └── img
        └── test.png
```

## ファイルの作成

`admin`ディレクトリ配下にファイルを作成。

```
$ touch admin.html
```

```html:admin.htmlの内容
<h1>This is Admin File</h1>
```

## 設定ファイルの変更

:::message
`192.168.192.1`は自分のPCのIPアドレスを入力してください。
:::

```diff:static.conf
server {
  listen 8000;

  server_name www.example.com;

  access_log /var/log/nginx/www.example.com_access.log;
  error_log /var/log/nginx/www.example.com_error.log;

  root /var/www/html;

  error_page 404 /404.html;
  error_page 500 502 503 504 /50x.html;

  location /images/ {
    alias /var/www/img/;
  }

  location /secret/ {
    # エラーページを応答する
    return 404;
  }

+  location /admin/ {
+    allow 192.168.192.1; # 自分のIPアドレスに変更
+    deny all;
+  }
}
```

## 動作確認

nginxの設定を適用させるコマンドを入力する。順次適用されていく。

```
(nginxコンテナ)# nginx -s reload
```

ブラウザで`localhost:8000/admin/admin.html`にアクセスすると、

`admin.html`の内容が表示される。

同一ネットワークの別のPCで`localhost:8000/admin/admin.html`にアクセスするか`allow 192.168.192.1;`のIPを変更すると、

403 Forbidden が表示される。

# この章の完了後のnginxの設定

:::details nginxの設定
```
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

    location /secret/ {
    # エラーページを応答する
    return 404;
  }

  location /admin/ {
    allow 192.168.192.1;
    deny all;
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
    │   ├── admin
    │   │   └── admin.html
    │   ├── docs
    │   │   └── sub.html
    │   ├── index.html
    │   └── secret
    │       └── secret-file.html
    └── img
        └── test.png
```
