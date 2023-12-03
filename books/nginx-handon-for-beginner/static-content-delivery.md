---
title: "HTMLファイル配信"
---
# 目的

 - nginxの設定の一部を理解する

# 動作イメージ


ブラウザから来たリクエストに応じたファイルを返却する。

図中の各数字はポートを意味する。

`63148`ポートについては、[エフェメラルポート](https://wa3.i-3-i.info/word15366.html)が参考になる。

![](/images/nginx-handon-for-beginner/static-content-delivery/html.drawio.png)

# HTMLファイルの作成

## ディレクトリの作成

まずは、ディレクトリを作成する。

```
$ mkdir -p nginx/html
$ cd nginx/html
```

```
.
├── docker-compose.yml
└── nginx
    └── html　<---- これを作成
```


## ファイルの作成

```
$ touch index.html
$ touch 404.html
$ touch 50x.html
```

エディタを使用して各ファイルを編集する。

```:index.htmlの内容
<h1>Hello Nginx</h1>
```

```:404.htmlの内容
<h1>File Not Found</h1>
```

```:50x.htmlの内容
<h1>Internal Server Error</h1>
```

説明のために別ディレクトリのファイルも作成する。

```
$ mkdir docs
$ cd docs
```

```
$ touch sub.html
```

`sub.html`を編集する。

```:sub.htmlの内容
<h1>This is Sub Docs</h1>
```

## 設定ファイルの追加

`nginx`ディレクトリに設定ファイルを追加する


```
$ mkdir conf.d
$ cd conf.d
```

```
.
├── docker-compose.yml
└── nginx
    ├── conf.d　<---- これを作成
    └── html
        ├── 404.html
        ├── 50x.html
        ├── docs
        │   └── sub.html
        └── index.html
```

作成した`nginx`ディレクトリに設定ファイルを作成

```
$ touch static.conf
```

```:static.confの内容
server {
  listen 8000;

  server_name www.example.com;

  access_log /var/log/nginx/www.example.com_access.log main;
  error_log /var/log/nginx/www.example.com_error.log;

  root /var/www/html;

  error_page 404 /404.html;
  error_page 500 502 503 504 /50x.html;
}
```

設定ファイルについて説明する。

```
  listen 8000;
```

で8000番ポートへのアクセスの設定であることを明示する。

```
  server_name www.example.com;
```

でサーバの名前を決めている。今回は`www.example.com`である。

```
  access_log /var/log/nginx/www.example.com_access.log main;
```

でサーバへのアクセスログを指定のパスに記載する。

```
  error_log /var/log/nginx/www.example.com_error.log;
```

でサーバへのエラーログを指定のパスに記載する。

```
  root /var/www/html;
```

でファイル配信するルートディレクトリを指定する。

```
  error_page 404 /404.html;
  error_page 500 502 503 504 /50x.html;
```

で404エラーの時は/404.html、500 502 503 504の時は/50x.htmlを表示する。


## nginxコンテナに作成したファイルを適用

`docker-compose.yml`ファイルを編集する。

```diff yml:docker-compose.ymlの内容
version: "3"
services:
  nginx:
    image: nginx:1.23.3
    container_name: nginx
    ports:
      - "8000:8000"
+    volumes:
+      - ./nginx/conf.d:/etc/nginx/conf.d
+      - ./nginx/html:/var/www/html
```

`docker-compose.yml`があるディレクトリに移動して、dockerを再起動する。

```
$ docker-compose down
$ docker-compose up
```

# 動作確認

nginxの設定を適用させるコマンドを入力する。順次適用されていく。

```
(nginxコンテナ)# nginx -s reload
```

## ルートへのアクセス

ブラウザで`localhost:8000`にアクセスする。

先ほど作成した`index.html`の内容が表示されている。

## ルート以外へのアクセス

ブラウザで`localhost:8000/docs/sub.html`にアクセスする。

先ほど作成した`sub.html`の内容が表示されている。

## 存在しないファイルへのアクセス

ブラウザで`localhost:8000/test.html`にアクセスする。

先ほど作成した`404.html`の内容が表示されている。

## サーバ側でエラー発生時

今回は再現しないが、500, 502, 503, 504のHTTPエラーは発生した場合、`50x.html`が表示される。

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
    └── html
        ├── 404.html
        ├── 50x.html
        ├── docs
        │   └── sub.html
        └── index.html
```
