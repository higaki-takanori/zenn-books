---
title: "gzip"
---

# 目的

- gzipを理解する

# gzip

gzipの説明は[こちら](https://wa3.i-3-i.info/word14219.html)が参考になる。

ファイルをそのまま渡すのではなく、圧縮して渡したほうが通信量が減って嬉しい。

今回は、ファイルサイズが大きいcssファイルを用意し、通信量が減っているかを確認する。

## cssファイルの作成

`styles`ディレクトリに`styles.css`ファイルを作成する

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
    │   ├── secret
    │   │   └── secret-file.html
    │   └── styles            # <----これを作成
    │       └── styles.css    # <----これを作成
    └── img
        └── test.png
```

容量を増やすために`styles.css`には無駄な記載が含まれています。

長いので折りたたみます。

:::details styles.cssの内容
```:styles.cssの内容
html {
  background-color: #000;
}

h1 {
  color: gray;
}

button {
  margin: 20px;
  height: 30px;
  width: 90px;
  border: 0;
  border-radius: 5px;
  background-color: darkblue;
  color: #fff;
}

.circle {
  background-color: rgb(0, 0, 0);
  width: 100px;
  height: 100px;
  border-radius: 50px;
  position: relative;
  z-index: -5;
  margin-left: 250px;
  top: 450px;
}

.circle2 {
  background-color: rgb(110, 110, 253);
  width: 100px;
  height: 100px;
  border-radius: 50px;
  z-index: -4;
  margin-left: 250px;
  top: 400px;
  position: relative;
}

.circle3 {
  background-color: rgb(14, 1, 87);
  width: 100px;
  height: 100px;
  border-radius: 50px;
  z-index: -3;
  margin-left: 250px;
  top: 350px;
  position: relative;
}

.circle4 {
  background-color: rgb(34, 55, 148);
  width: 100px;
  height: 100px;
  border-radius: 50px;
  z-index: -2;
  margin-left: 250px;
  top: 300px
}

.circle5 {
  background-color: rgb(46, 35, 211);
  width: 100px;
  height: 100px;
  border-radius: 50px;
  z-index: -1;
  margin-left: 250px;
  top: 250px;
  position: relative;
}

.box1 {
  background-color: rgba(255, 0, 64, 0.548);
  width: 200px;
  height: 200px;
  border: 2px solid #000;
  padding: 20px;
}

.box2 {
  background-color: green;
  width: 200px;
  height: 200px;
  border: 2px solid #000;
  margin: 20px;
}

.box3 {
  background-color: red;
  width: 200px;
  height: 200px;
  border: 20px solid #000;
}

.box4 {
  background-color: grey;
  width: 200px;
  height: 200px;
  border: 20px solid #000;
  padding: 20px;
  border-radius: 20px;
}

.box {
  width: 100px;
  height: 100px;
  background-color: blue;
  margin: 10px;
  border-radius: 20px;
}

.container {
  width: 200px;
  height: 600px;
  background-color: orange;
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  justify-content: space-evenly;
}

.container2 {
  width: 500px;
  height: 500px;
  padding: 5px;
  background-color: red;
  display: flex;
  justify-content: flex-end;
}

.container3 {
  width: 500px;
  height: 500px;
  padding: 5px;
  background-color: rgb(87, 87, 87);
}

.container4 {
  width: 600px;
  height: 600px;
  display: flex;
  justify-content: center;
  flex-direction: column;
}

.child2 {
  background-color: lightblue;
  width: 200px;
  height: 100px;
  border-radius: 20px;
}
```
:::

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

+    gzip on;
+    gzip_types text/css text/javascript
+              application/x-javascript application/javascript
+              application/json;
+    gzip_min_length 1k;
+    gzip_disable "msie6";
  }
}
```

これにより、`/styles/styes.css`がgzipされて返却される。

## 動作確認

### gzip適用前

`localhost:8000`にアクセスする。

![](/images/nginx-handon-for-beginner/gzip/before_gzip.png)


`styles.css`の容量は`2.5kB`ほどである。

### gzip適用後

nginxの設定を適用させるコマンドを入力する。順次適用されていく。

```
(nginxコンテナ)# nginx -s reload
```

キャッシュなどを削除して`localhost:8000`にアクセスする。

![](/images/nginx-handon-for-beginner/gzip/after_gzip.png)

`styles.css`の容量は`868B`ほどになっている。

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

    gzip on;
    gzip_types text/html text/css text/javascript
              application/x-javascript application/javascript
              application/json;
    gzip_min_length 1k;
    gzip_disable "msie6";
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
    │   ├── secret
    │   │   └── secret-file.html
    │   └── styles
    │       └── styles.css
    └── img
        └── test.png
```
