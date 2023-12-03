---
title: "try_files"
---

# 目的

- try_filesを理解する

# try_files

try_filesについて説明する。

[公式ドキュメント](http://nginx.org/en/docs/http/ngx_http_core_module.html#try_files)は以下である。

>>Syntax:	try_files file ... uri;
try_files file ... =code;
Default:	—
Context:	server, location

>Checks the existence of files in the specified order and uses the first found file for request processing; the processing is performed in the current context. The path to a file is constructed from the file parameter according to the root and alias directives. It is possible to check directory’s existence by specifying a slash at the end of a name, e.g. “$uri/”. If none of the files were found, an internal redirect to the uri specified in the last parameter is made.

まとめると、

- tyr_filesの後に記載されている fileやuriを左から順に検索する
- ファイルがない場合は内部リダイレクトする
- 何も見つからなかった場合はcodeを返却する

try_filesの挙動については以下が参考になる。

(参考)[https://gist.github.com/kenjiskywalker/4596258]

## htmlファイルの作成

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
    │   │   ├── index.html <---- これを作成
    │   │   └── sub.html
    │   ├── index.html
    │   ├── secret
    │   │   └── secret-file.html
    │   └── styles
    │       └── styles.css
    └── img
        └── test.png
```

```:docs/index.htmlの内容
<h1>This is Docs Page</h1>
```

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

+  location /docs/ {
+    try_files $uri $uri/ $uri.html =404;
+  }


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

## 動作確認

nginxの設定を適用させるコマンドを入力する。順次適用されていく。

```
(nginxコンテナ)# nginx -s reload
```

ブラウザで`localhost:8000/docs/sub`にアクセスすると、`/docs/sub.html`の内容が表示される。

![](/images/nginx-handon-for-beginner/tryfiles/sub.png)

---

この挙動について詳しく説明する。

:::details $uriの確認方法
確認後は元に戻す。
```diff:static.confを
  location /docs/ {
+    add_header Content-Type text/plain;
+    return 200 "document_root: $document_root, request_uri: $request_uri, uri: $uri, is_args: $is_args, args: $args";
-    try_files $uri $uri/ $uri.html =404;
  }
```

![](/images/nginx-handon-for-beginner/tryfiles/uri.png)
:::

1. `location /docs/`に合致するため`try_files $uri $uri/ $uri.html =404;`が実行

2. `$uri`の検索
   1. $uriが`/docs/sub`のため、`/docs/sub`というファイルがあるかを確認
   2. `/docs/sub`というファイルがなかったため、内部リダイレクト（内部リダイレクト時、呼び出し元の`location /docs/`は通らなさそう。この辺は要調査）
   3. `location /`に合致するが、返却されるものがない

3. `$uri/`の検索
   1. `/docs/sub/(index.html)`というファイルがあるかを確認(`/`ディレクトリ検索時は`/index.html`が検索される)
   2. `/docs/sub/index.html`がないため、内部リダイレクト
   3. `location /`に合致するが、返却されるものがない

4. `$uri.html`の検索
   1. `/docs/sub.html`というファイルがあるか確認
   2. ファイルがあるため、`/docs/sub.html`を返却


色々遊んでみたい人は`/docs/index.html`を作ったり、`location /`で何か返却するように変更してみると良いかも

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

  location /docs/ {
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
