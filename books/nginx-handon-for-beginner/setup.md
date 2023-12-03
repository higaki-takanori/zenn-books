---
title: "環境構築"
---

# 前提条件

- dockerがインストールされている

# 動作確認

## nginxハンズオンディレクトリを作成

```
$ mkdir nginx-hands-on
$ cd nginx-hands-on
```

## docker-compose.ymlの作成

```
$ touch docker-compose.yml
```

```:docker-compose.ymlの内容
version: "3"
services:
  nginx:
    image: nginx:1.23.3
    container_name: nginx
    ports:
      - "8000:8000"
```

## dockerの立ち上げ

nginxのコンテナを立ち上げる。

```
$ docker-compose up
```

## nginxの設定確認

nginxコンテナのシェルに入る。

```
$ docker-compose exec nginx bash
```

nginxの設定を確認する。

```
(nginxコンテナ)# nginx -T
```

以下のような表示になっているはず。

```:表示
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
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
    text/css                                         css;
...省略
    video/x-ms-wmv                                   wmv;
    video/x-msvideo                                  avi;
}

# configuration file /etc/nginx/conf.d/default.conf:
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```

1行目に

```:1行目の表示
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
```

とあるように

`/etc/nginx/nginx.conf`に設定ファイルがありそう、、、

実際に確認してみる。

```
(nginxコンテナ)# cat /etc/nginx/nginx.conf
```

```:nginx.confの内容
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
```

予想通りあった。

先ほど確認した設定の先頭と同じ内容が表示された。

## 各設定ファイルの読み込む場所

また、`/etc/nginx/nginx.conf`の最後から2行目に

```:/etc/nginx/nginx.conf の最後から2行目
    include /etc/nginx/conf.d/*.conf;
```

とあるように、nginxの各設定ファイルは`/etc/nginx/conf.d/`配下の`.conf`ファイルも追加で読み込む記載がしてある。

実際に何があるか確かめてみる。

```
(nginxコンテナ)# ls /etc/nginx/conf.d/
```

```:表示
default.conf
```

`default.conf`が存在していた。

中身を確認してみる。

```
(nginxコンテナ)# cat /etc/nginx/conf.d/default.conf
```

```:default.confの内容
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```

これは先ほど確認したnginxの設定の最後に記載されていた内容と同じである。

`/etc/nginx/conf.d/`配下の`.conf`ファイルも追加で読み込んでいることを確認できた。

つまり、nginxの設定は`http`の中に複数の`server`が存在するイメージになる。

```:nginxの設定のイメージ
...省略

http {
  ...省略

  server {
    listen       XXXX;
    ...省略
  }

  server {
    listen       YYYY;
    ...省略
  }
}
```

プロジェクトによっては、`include`文を追加して各設定ファイルの読み込むディレクトリを追加しても良さそう。

# この章の完了後のディレクトリ構造

```
.
└── docker-compose.yml
```
