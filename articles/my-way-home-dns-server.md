---
title: "僕はこうやって自宅DNSサーバを使ってます"
emoji: "📛"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Bind9", "DNS", "Github Actions"]
published: true
---

# はじめに

自宅にProxmoxサーバをたてて、しばらくの間IPアドレス直打ちしていました。

なんでワイがIPアドレスなんぞ調べなあかんのや！！の気持ちが強くなってきました。

そこで、DNSサーバを建てることにしました。

脳内に何も記憶したくなかったので、Github上で設定を管理して、Githubのプルリク経由で自動で設定が変更されるようにしました。

備忘録として、どうやって構築していったかを記録します。

:::message alert

本記事ではGithub上にDNS設定ファイルをおきます。

全て自己責任でお願いします。

:::

# 自宅環境

- ルータ: YAMAHA RTX1210
  - YAMAHA RTX1210で簡易的なDNSサーバ構築できるけど、別でDNSサーバを構築しました。
- DNSサーバ: ProxmoxのCT上でBind9 

# 結論

Github ActionsのセルフホステッドランナーでDNSサーバの設定を変更させる！

# Special Thanks

[デロさん](https://x.com/dero1to)

デロさんにGithub Actionsのセルフホステッドランナー+DNSサーバについて教えていただきました！

# やり方

以下の構築・設定変更が必要になります。

- DNSサーバの構築
- ルータ（YAMAHA RTX1210）

## DNSサーバの構築

### Proxmox上にDNSサーバ用のCT（VM）を建てる

Proxmox上にDNSサーバ用のCTを建ててください。 

既存CTやVMを使用してもかまいません。

### Bind9のインストール

まずは以下のコマンドでBind9をインストールします。

```shell
sudo apt install bind9
```

インストールが完了すると、`/etc/bind`が作成されます。

次に、自分はDNS設定をGithub Actionsで変更したいので、`/etc/bind`配下をgitの管理下にします。

```shell
git init
git add .
git commit -m "init commit"
```

後は適当なprivateリポジトリにpushしておいてください。

### DNSにゾーン追加

:::message
今回の記事では設定内容を以下とします。（ご自身の環境に合わせてください）

- 自宅内のドメイン名：`hoge.internal`
- ゾーンファイル：`zones.internal`
- 正引きゾーン定義ファイル：`db.internal`
- 逆引きゾーン定義ファイル：`0.168.192.in-addr.arpa`
- DNSサーバのIPアドレス：`192.168.0.9`
- 名前解決したいサーバ
  - IPアドレス：`192.168.0.5`
  - 設定したいサブドメイン：`fuga`
    - このサーバには`fuga.hoge.internal`でアクセスできるようしたい
:::

#### 正引き

まずは、ゾーン定義ファイル`/etc/bind/db.internal`を以下の内容で作成します。

```
$TTL    604800
@       IN      SOA     ns1.hoge.internal. admin.hoge.internal. ( // 要変更
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800         ; Negative Cache TTL
)

@               IN      NS      ns1.hoge.internal. // 要変更
@               IN      A       192.168.0.9 // 要変更 サブドメインなしでアクセスした時のIPアドレス
ns1             IN      A       192.168.0.9 // 要変更 DNSサーバのIPアドレス
fuga // 要変更   IN      A       192.168.0.5 // 要変更 割り当てたいサーバのIPアドレス
```

次に`zones.internal`を以下の内容で作成します。

```
zone "hoge.internal" { // 要変更
    type master;
    file "/etc/bind/db.internal"; // 要変更
    allow-update { none; };
};
```

最後に`/etc/bind/named.conf.local`を変更します。

```
include "/etc/bind/zones.internal"; //追加
```

DNS設定の文法の確認を以下のコマンドで行ってください。

```shell
named-checkconf
```

うまくいっていれば、何も出力されないです。

次に実際に名前解決してみましょう。

まずはゾーン情報をリロードします。

```
sudo rndc reload
```

次に、`dig`コマンドで名前解決を行います。（DNSサーバのIPアドレスはご自身の環境のものに変更してください）

```shell
dig @<DNSサーバのIPアドレス> <IPアドレスが知りたいドメイン名>

dig @192.168.0.9 fuga.hoge.internal
```

これでDNSに正引きゾーンの追加が完了しました。

自分はここで`git push`します。

#### 逆引き

ゾーン定義ファイル`/etc/bind/0.168.192.in-addr.arpa`を以下の内容で作成します。

```
$TTL    604800
@       IN      SOA     ns1.hoge.internal. admin.hoge.internal. ( // 要変更
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800         ; Negative Cache TTL
)

@               IN      NS      ns1.hoge.internal. // 要変更
9               IN      PTR     ns1.hoge.internal. // 要変更 
5               IN      PTR     fuga.hoge.internal. // 要変更
```

次に`zones.internal`に以下の内容を追記します。

```
zone "0.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/0.168.192.in-addr.arpa";
    allow-update { none; };
};
```

DNS設定の文法の確認を以下のコマンドで行ってください。

```shell
named-checkconf
```

うまくいっていれば、何も出力されないです。

次に実際に名前解決してみましょう。

まずはゾーン情報をリロードします。

```
sudo rndc reload
```

次に、`dig`コマンドで逆引きの名前解決を行います。（DNSサーバのIPアドレスはご自身の環境のものに変更してください）

```shell
dig @<DNSサーバのIPアドレス> -x <ドメイン名が知りたいIPアドレス>

dig @192.168.0.9 192.168.0.5
```

これでDNSに逆引きゾーンの追加が完了しました。

自分はここで`git push`します。

### Github Actions のセルフホステッドランナーの構築

Github Actionsは[こちら](https://qiita.com/h_tyokinuhata/items/7a9297f75d0513572f4a)を参考に構築してください。

### Github Actionsのworkflowsの設定

workflowsはgit管理しているので、DNSサーバの外で`git clone`して設定しても良いです！

ここではDNSサーバ内を前提に説明します。

まず、`/etc/bind`配下に`.github/workflows`ディレクトリを作成してください。

次に、`deploy.yaml`を以下の内容で保存してください。

```yaml
name: Release DNS Setting
on:
  pull_request:
    types: [closed]
    branches:
      - main

jobs:
  deploy-dns-setting:
    name: Deploy
    runs-on: [self-hosted, linux]
    if: github.event.pull_request.merged == true
    steps:
      - name: deploy
        working-directory: /etc/bind
        run: |
          sudo git pull origin main
          sudo rndc reload
```

これで、DNS設定を変更したいときは`main`ブランチへのプルリクをマージするだけで設定が変更されるようになります。

簡単な流れは以下です。

- `main`ブランチへのプルリクをマージ
- Github Actionsセルフホステッドランナーがworkflowsに沿って動き始める
  - DNSの設定変更

最終的なdirectory構成だけ示しておきます。

```
/etc/bind
├── .git
│   └── ...
├── .github
│   └── workflows
│       ├── README.md
│       └── deploy.yaml
├── bind.keys
├── db.0
├── db.127
├── 0.168.192.in-addr.arpa // 追加
├── db.255
├── db.empty
├── db.internal // 追加
├── db.local
├── named.conf
├── named.conf.default-zones
├── named.conf.local
├── named.conf.options
├── rndc.key
├── zones.internal // 追加
└── zones.rfc1918
```

## ルータの設定変更

次に、自宅ネットワークで`*.hoge.internal`にアクセスする時は、先ほど構築したDNSサーバを使用して名前解決を行うようにルータの設定を変更します。

まず、YAMAHAルータにCLIで接続します。

```shell
telnel <YAMAHAルータのIPアドレス>
```

次に以下の設定を追加します。

```
dns server select <DNS サーバー選択テーブルの番号> <DNSサーバのIPアドレス> any <解決したいドメイン>
dns server select <DNS サーバー選択テーブルの番号> <DNSサーバのIPアドレス> any <逆引き>
```

例

```
dns server select 500001 192.168.0.9 any .hoge.internal
dns server select 500002 192.168.0.9 any 0.168.192.in-addr.arpa
```

※ [参考](https://www.rtpro.yamaha.co.jp/RT/manual/nvr500/dns/dns_server_select.html)

これで、自宅ネットワークで`*.hoge.internal`にアクセスした時、名前解決されます。

# まとめ

自宅DNSサーバを構築した時の備忘録を記事に残しました。

自分的にはDNSのゾーン定義ファイルとかGithub Actionsセルフホステッドランナーを触れて楽しめました。

みなさまのDNSサーバの管理方法をコメントで教えていただけますと嬉しいです！！！

# 参考

- [GitHub Actionsのセルフホステッドランナーを構築する](https://qiita.com/h_tyokinuhata/items/7a9297f75d0513572f4a)
- [20.9 DNS 問い合わせの内容に応じた DNS サーバーの選択](https://www.rtpro.yamaha.co.jp/RT/manual/nvr500/dns/dns_server_select.html)


