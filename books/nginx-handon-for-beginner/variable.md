---
title: "条件分岐・変数"
---

# 目的

- 条件分岐を理解する
- 変数を理解する

# 条件分岐

iPhoneからアクセス時は表示するHTMLを変更する。
今回はデジタル庁のHPに変更する。

## 設定ファイルの変更

```diff:static.conf
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
    allow 192.168.192.1; # 自分のIPアドレスに変更
    deny all;
  }

+  location / {
+    if ($http_user_agent ~ "iPhone") {
+      return 301 https://www.digital.go.jp/;
+    }
+  }
}
```

これにより、iPhoneから`/`のリクエストはデジタル庁のHPを返却するようになる。

## 動作確認

nginxの設定を適用させるコマンドを入力する。順次適用されていく。

```
(nginxコンテナ)# nginx -s reload
```

1. iPhoneとPCを同一ネットワークにする。デザリングなど
2. iPhoneのブラウザで`192.168.192.1(PCのIPアドレス)`にアクセスすると、デジタル庁のHPが表示される。

# 変数

iPadやiPodからアクセス時も同様に表示するHTMLを変更する。

nginxのデフォルトで用意されている変数は以下に記載されている。
[公式サイト](https://nginx.org/en/docs/varindex.html)

今回は、`$http_user_agent`はデフォルト変数だが、`$redirect_to_mobile`は自作変数である。

公式サイトによると、`$http_(ヘッダーフィールド)`で任意のヘッダーフィールドを取得できる。

> $http_name
arbitrary request header field; the last part of a variable name is the field name converted to lower case with dashes replaced by underscores

今回は、ヘッダーフィールドが`user_agent`の情報を取得している。

![](/images/nginx-handon-for-beginner/variable/2023-09-16_13.53.28.png)

## 設定ファイルの変更

```diff:static.conf
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
    allow 192.168.192.1; # 自分のIPアドレスに変更
    deny all;
  }

  location / {
-    if ($http_user_agent ~ "iPhone") {
-      return 301 https://www.digital.go.jp/;
-    }
+    set $redirect_to_mobile 0;

+    if ($http_user_agent ~ "iPhone") {
+      set $redirect_to_mobile 1;
+    }
+    if ($http_user_agent ~ "iPod") {
+      set $redirect_to_mobile 1;
+    }
+    if ($http_user_agent ~ "iPad") {
+      set $redirect_to_mobile 1;
+    }

+    if ($redirect_to_mobile) {
+      return 301 https://www.digital.go.jp/;
+    }
  }
}
```

上記の変更により、

(PCの場合)
変数`$redirect_to_mobile`は0のままで、
`if ($redirect_to_mobile)`の条件が偽となるため、`/index.html`を返却する。

(iPhone、iPad、iPodの場合)
変数`$redirect_to_mobile`は1になり、
`if ($redirect_to_mobile)`の条件が真となるため、`https://www.digital.go.jp/;`にリダイレクトされる。

## 動作確認

nginxの設定を適用させるコマンドを入力する。順次適用されていく。

```
(nginxコンテナ)# nginx -s reload
```

1. iPadとPCを同一ネットワークにする。デザリングなど
2. iPadのブラウザで`192.168.192.1(PCのIPアドレス)`にアクセスすると、デジタル庁のHPが表示される。

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
    allow 192.168.192.1; # 自分のIPアドレスに変更
    deny all;
  }

  location / {
    set $redirect_to_mobile 0;

    if ($http_user_agent ~ "iPhone") {
      set $redirect_to_mobile 1;
    }
    if ($http_user_agent ~ "iPod") {
      set $redirect_to_mobile 1;
    }
    if ($http_user_agent ~ "iPad") {
      set $redirect_to_mobile 1;
    }

    if ($redirect_to_mobile) {
      return 301 https://www.digital.go.jp/;
    }
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
