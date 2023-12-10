---
title: "php-fpm(UNIXドメインソケット)"
---

# 目的

- UNIXドメインソケット通信によるphp-fpmについて理解する

# 動作イメージ

nginxとphp-fpmコンテナ間はunixドメインソケット通信が行われている。

ただし、これはDockerのマウントによってソケットファイルを共有できるため、動作するだけである。

本来は別サーバであれば、UNIXドメインソケットはできない。

unixドメインソケットは[こちら](https://qiita.com/kuni-nakaji/items/d11219e4ad7c74ece748)が参考になる。

![](/images/nginx-handon-for-beginner/php-fpm-unix/php-fpm-unix.drawio.png)

:::message alert
今回はDockerの作法に則り、別コンテナに分割しているが、
コンテナを使用しない場合は、一つのサーバにnginxとphp-fpmを動作させるのが良さそう。
:::

本来は、以下のように1つのサーバ内で処理させるのが良さそうだが、
今回はDokcerを使用しているため、別コンテナに分けている。

![](/images/nginx-handon-for-beginner/php-fpm-unix/unix-same.drawio.png)

## php-fpm(unixドメイン通信)の設定ファイルの作成

php-fpm(unixドメイン通信)の設定ファイルを作成する。

これはphp-fpmコンテナで使用するためのファイルである。

```
.
├── docker
│   └── nginx
├── docker-compose.yml
├── nginx
│   ├── conf.d
│   │   └── static.conf
│   ├── html
│   │   ├── 404.html
│   │   ├── 50x.html
│   │   ├── admin
│   │   │   └── admin.html
│   │   ├── docs
│   │   │   └── sub.html
│   │   ├── index.html
│   │   ├── secret
│   │   │   └── secret-file.html
│   │   └── styles
│   │       └── styles.css
│   └── img
│       └── test.png
└── php
    └── php-fpm.d
        ├── php-fpm.conf
        └── unix-socket.conf  <------ これを作成する
```

```:unix-socket.confの内容
[www]
; [www] のlistenを上書き
listen = /var/run/php-fpm.sock
listen.owner = www-data
listen.group = www-data
;listen.mode = 0660
```

こちらについて、説明する。

まず、phpコンテナ内に入る。


```
$ docker compose exec php bash
```

php-fpmの設定を確認する。

```
(phpコンテナ)# ls /usr/local/etc/php-fpm.conf
```

```:/usr/local/etc/php-fpm.confの内容
...省略

;;;;;;;;;;;;;;;;;;;;
; Pool Definitions ;
;;;;;;;;;;;;;;;;;;;;

; Multiple pools of child processes may be started with different listening
; ports and different management options.  The name of the pool will be
; used in logs and stats. There is no limitation on the number of pools which
; FPM can handle. Your system will tell you anyway :)

; Include one or more files. If glob(3) exists, it is used to include a bunch of
; files from a glob(3) pattern. This directive can be used everywhere in the
; file.
; Relative path can also be used. They will be prefixed by:
;  - the global prefix if it's been set (-p argument)
;  - /usr/local otherwise
include=etc/php-fpm.d/*.conf
```

`include=etc/php-fpm.d/*.conf`とある。

ここに詳細な設定ファイルがありそうなので、ディレクトリ内を確認する。

```
(phpコンテナ)# ls /usr/local/etc/php-fpm.d
```

```:表示
docker.conf  www.conf  www.conf.default  zz-docker.conf
```

その中で、`zz-docker.conf`の内容を確認すると、
```
[global]
daemonize = no

[www]
listen = 9000
```

となっている。

UNIXドメインソケット通信にするためには、
php-fpmの[www]は`listen = /var/run/php-fpm.sock`に変更したい。

そこで、`zz-docker.conf`の設定を上書きするために、`zzz-unix-socket.conf`を作成する。

## docker-compose.ymlの変更

```diff:docker-compse.ymlの修正
version: "3"
services:
  nginx:
    image: nginx:1.23.3
    container_name: nginx
    ports:
      - "8000:8000"
      - "8001:8001"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/html:/var/www/html
      - ./nginx/img:/var/www/img
      - ./php/src:/var/www/php
+     - ./socket:/var/run
  php:
    image: php:8.2-fpm
    container_name: php-fpm
    ports:
      - "9000"
    volumes:
      - ./php/src:/var/www/php
      - ./php/php-fpm.d/php-fpm.conf:/usr/local/etc/php-fpm.d/www.conf
+     # /usr/local/etc/php-fpm.d/zz-docker.conf の内容を上書きするために、zzzを頭文字につける
+     - ./php/php-fpm.d/unix-socket.conf:/usr/local/etc/php-fpm.d/zzz-unix-socket.conf
+     - ./socket:/var/run
```

ここで、nginxとphpコンテナのどちらも`./socket:/var/run`をvolumesすることで、

ソケットファイルを共有している。

## nginxの設定ファイルの変更

`php.conf`のtcpソケット通信の部分をunixドメインソケット通信に変更する。

```diff:php.confの内容
server {
  listen 8001;

  server_name php.example.com;

  access_log /var/log/nginx/php.example.com_access.log main;
  error_log /var/log/nginx/php.example.com_error.log;

  location / {
    try_files $uri $uri/ /index.php?$query_string;
  }

  location ~ \.php$ {
    root /var/www/php;

    include fastcgi_params;
    fastcgi_index           index.php;
    fastcgi_param           SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
    fastcgi_param           DOCUMENT_ROOT $realpath_root;
-   fastcgi_pass            php:9000;
+   fastcgi_pass            unix:/var/run/php-fpm.sock;
  }
}
```

## 動作確認

dockerコンテナを再起動する。

```
$ docker-compose down
$ docker-compose up
```

`localhost:8001`にアクセスする。

![](/images/nginx-handon-for-beginner/php-fpm-tcp/indexphp.png)

`localhost:8001/?foo=123`にアクセスする。

![](/images/nginx-handon-for-beginner/php-fpm-tcp/foo.png)

表示されているリンクか`localhost:8001/phpinfo.php`にアクセスする。

![](/images/nginx-handon-for-beginner/php-fpm-tcp/phpinfo.png)

# この章の完了後のnginxの設定

:::details nginxの設定
```
...省略
# configuration file /etc/nginx/conf.d/php.conf:
server {
  listen 8001;

  server_name php.example.com;

  access_log /var/log/nginx/php.example.com_access.log main;
  error_log /var/log/nginx/php.example.com_error.log;

  location / {
    try_files $uri $uri/ /index.php?$query_string;
  }

  location ~ \.php$ {
    root /var/www/php;

    include                 fastcgi_params;
    fastcgi_index           index.php;
    fastcgi_param           SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
    fastcgi_param           DOCUMENT_ROOT $realpath_root;
    fastcgi_pass            unix:/var/run/php-fpm.sock;
  }
}
# configuration file /etc/nginx/fastcgi_params:

fastcgi_param  QUERY_STRING       $query_string;
fastcgi_param  REQUEST_METHOD     $request_method;
fastcgi_param  CONTENT_TYPE       $content_type;
fastcgi_param  CONTENT_LENGTH     $content_length;

fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;
fastcgi_param  REQUEST_URI        $request_uri;
fastcgi_param  DOCUMENT_URI       $document_uri;
fastcgi_param  DOCUMENT_ROOT      $document_root;
fastcgi_param  SERVER_PROTOCOL    $server_protocol;
fastcgi_param  REQUEST_SCHEME     $scheme;
fastcgi_param  HTTPS              $https if_not_empty;

fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;

fastcgi_param  REMOTE_ADDR        $remote_addr;
fastcgi_param  REMOTE_PORT        $remote_port;
fastcgi_param  SERVER_ADDR        $server_addr;
fastcgi_param  SERVER_PORT        $server_port;
fastcgi_param  SERVER_NAME        $server_name;

# PHP only, required if PHP was built with --enable-force-cgi-redirect
fastcgi_param  REDIRECT_STATUS    200;

# configuration file /etc/nginx/conf.d/static.conf:
server {
  listen 8000;

  server_name www.example.com;

  access_log /var/log/nginx/www.example.com_access.log main;
  error_log /var/log/nginx/www.example.com_error.log;

  root /var/www/html;

  error_page 404 /404.html;
  error_page 500 502 503 504 /50x.html;

  location /docs/ {
    # add_header Content-Type text/plain;
    # return 200 "document_root: $document_root, request_uri: $request_uri, uri: $uri, is_args: $is_args, args: $args";
    try_files $uri $uri/ $uri.html =404;
  }

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
    gzip_types text/css text/javascript
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
├── docker
│   └── nginx
├── docker-compose.yml
├── nginx
│   ├── conf.d
│   │   ├── php.conf
│   │   └── static.conf
│   ├── html
│   │   ├── 404.html
│   │   ├── 50x.html
│   │   ├── admin
│   │   │   └── admin.html
│   │   ├── docs
│   │   │   └── sub.html
│   │   ├── index.html
│   │   ├── secret
│   │   │   └── secret-file.html
│   │   └── styles
│   │       └── styles.css
│   └── img
│       └── test.png
└── php
    ├── php-fpm.d
    │   └── php-fpm.conf
    └── src
        ├── index.php
        └── phpinfo.php
```
