---
title: "ネットワーク"
emoji: ""
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["network", "yamaha-router", "ntopng"]
published: false
publication_name: "levtech"
---

# はじめに

家のネットワークを見てぇな

俺は誰と通信してんだが知りたくなった

# 対象読者

- PC以外の通信を知りたい人
- スマホやAmazon Echoなどの機器の通信まで見たい人
  - （実際の通信の内容は暗号化されていて見えません）

# 準備するもの

- ポートミラーリングができるルータ
  - 筆者は 「YAMAHA ルータ RTX1210」 を購入しました。
- Proxmox VEがインストールされているPC
- Proxmox VEがインストールされているPCにNICが2つない場合はNIC
  - 筆者は miniPCだったので[UBSタイプのNIC](https://amzn.asia/d/cdHggA8)を購入しました。

# 注意事項

Proxmox VEについては説明しません。

詳しく知りたい方は[公式サイト](https://www.proxmox.com/en/)やこちらの[本](https://techbookfest.org/product/8HATSEF31zuZAkeTAFe1m6?productVariantID=xpgxNEK7Tc4kdFdBDwBBCu)がおすすめです。

# どんな感じ？

[//]: # (FIXME 画像とか貼り付ける)

こんな感じで誰と通信してるのか認識できて面白い！！！

自分はこんな通信してたんだを知ることができてテンション上がりました！！

# 環境

使用した機器類

- PC1: Macbook Pro
- PC2: Proxmox VEがインストールされているPC
- ルータ: RTX1210

筆者のプロバイダ契約

- V6プラス(IPoE方式)

# 導入

## RTX1210

まずはRTX1210の設定を変えていきます。

**RTX1210にアクセス**

MacPCとRTX1210のLAN1をLANケーブルで接続します。

MacのターミナルでRTX1210に接続します。

```shell
(Mac) $ telnet 192.168.100.1
```
※デフォルトだとRTX1210のIPアドレスは`192.168.100.1`に設定されています。

※パスワードを聞かれますが、デフォルトでは設定されていないので、「Enter」で大丈夫です。

**RTX1210の初期状態の確認**

RTX1210にアクセス後`show config`で設定を確認できます。

```shell:初期のRTX1210の設定
(RTX1210) > show config
# RTX1210 Rev.14.01.05 (Thu Nov  6 08:45:55 2014)
# MAC Address : ********
# Memory 256Mbytes, 3LAN, 1BRI
# main:  RTX1210 ver=00 serial=****** MAC-Address=*****
ess=***** MAC-Address=*****
# Reporting Date: Nov 3 16:04:53 2024
ip lan1 address 192.168.0.1/24
switch control use lan1 on terminal=on
switch control use lan2 on terminal=on
switch control use lan3 on terminal=on
ip filter 500000 restrict * * * * *
dhcp service server
dhcp server rfc2131 compliant except remain-silent
dhcp scope 1 192.168.0.2-192.168.0.191/24
dns host lan1
dns private address spoof on
schedule at 1 */Wed 00:00:00 * ntpdate ntp.nict.jp syslog
dashboard accumulate traffic on
```

**RTX1210のFWアップデート**

初期状態だとIPv4 over IPv6のトンネリングの設定ができないのでFWをアップデートする必要があります。

!!!!注意!!!!!

[公式サイト](https://network.yamaha.com/support/rtx1210_boot/)にあるように、特定の製造番号のRTX1210は対策しないとFWアップデート後に起動しなくなる可能性があるので注意してください
>弊社製ギガアクセスVPNルーター「RTX1210」におきまして、ファームウェア更新後に起動しなくなるなどの不具合が発生することが判明いたしました。この不具合は、本記事に掲載する対策を施すことで回避することができます。

FWのアップデート方法は[こちらのPDF](https://www.rtpro.yamaha.co.jp/RT/manual/rtx1210/Users.pdf)を参考に行なってください。
筆者は外部メモリでアップデートしました。

**RTX1210の設定変更**

デフォルトデートウェイ

NAT

DNS

DHCP

## ntopng

# 参考

[ぐえたんの書庫 ntopngをセットアップして自宅のネットワークトラフィックを監視する (proxmoxのVM)](https://guetan.dev/setup-ntopng/#rtx1200%E3%81%A7port-mirroring%E3%81%AE%E8%A8%AD%E5%AE%9A)