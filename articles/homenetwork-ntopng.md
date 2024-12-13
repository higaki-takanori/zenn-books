---
title: "ポートミラーリング + Wiresharkで家の通信見てみた"
emoji: "🔎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["network", "yamaha-router"]
published: false
publication_name: "levtech"
---

この記事は「[レバテック開発部 Advent Calendar 2024](https://qiita.com/advent-calendar/2024/levtech)」の 16 日目の記事です！

昨日の記事は、[kima](https://zenn.dev/kima000) さんが担当していました！

## はじめに

家のネットワーク通信が見たいなーの気持ちが抑えられなくなりました。

そこで、どうにかして見れないかを試してみた記事です。

## 対象読者

- ネットワーク興味がある方
- オンライン対応のゲーム機やAmazon Echoなどの機器の通信を見たい人
  - （実際の通信データは暗号化されていて見えません）

## どんな感じ？

![](/images/homenetwork-ntopng/switch-net.jpg)
*ゲーム機のIPアドレス*
![](/images/homenetwork-ntopng/wireshark-switch.png)
*Wiresharkで通信が見えている*

誰と通信してるのか認識できていい感じです！！！

自分はこんな通信してたんだを知ることができてテンション上がりました！！

馴染みのある通信方法でなんだか安心しました！！！


## 準備するもの

- ポートミラーリングができるルータ
- 無線LANルータ
- ルータ設定用のPC
- 通信確認用PC
  - ルータ設定用のPCと同じでも大丈夫です。
- (任意)通信確認用PCにLANポートがない場合はLANポート付きのNIC
  - 筆者は[UBSタイプのNIC](https://amzn.asia/d/cdHggA8)を購入しました。

## 環境

**使用した機器類**

以下の固有名詞で説明します。

- ポートミラーリングができるルータ: **RTX1210**
  - 工場出荷状態でした。
  - 筆者はヤフオクで購入しました。
- 無線LANルータ: **I-O DATA WN-DX1200GR**
- ルータ設定用PC: **Macbook Pro**
- 通信確認用PC: **Macbook Pro**

![](https://network.yamaha.com/var/site/storage/images/_aliases/size_large/1/8/8/8/18881-17-jpn-JP/rtx1210_main.jpg =250x)
*[引用：ヤマハネットワーク製品公式](https://network.yamaha.com/products/routers/rtx1210/spec#tab)*

**筆者のプロバイダ契約**

- V6プラス(IPoE方式)

**最終的なネットワーク構成**

![](/images/homenetwork-ntopng/home-network.webp)
*ネットワーク構成*

RTX1210の各LANの使用用途
- LAN1: 家とネットワーク機器と接続
- LAN2: インターネットと接続
- LAN3: 未使用

## 注意事項

:::message alert

通信で得た情報を悪用すると[不正アクセス禁止法](https://www.gmo.jp/security/security-all/information-security/blog/unauthorized-computer-access-law/)に触れる可能性があります！
使い方には十分ご注意ください。

:::

## RTX1210の導入

### RTX1210にアクセス

まずは、MacとRTX1210のLAN1の任意のポートをLANケーブルで接続します。

次に、MacのターミナルでRTX1210に接続します。

```shell
(Mac) $ telnet 192.168.100.1
```
※デフォルトだとRTX1210のIPアドレスは`192.168.100.1`に設定されています。

※パスワードを聞かれますが、デフォルトでは設定されていないので、「Enter」で通ります。

### RTX1210の初期状態の確認

RTX1210にアクセス後`show config`で設定を確認できます。

```shell:初期のRTX1210の設定
(RTX1210) > show config
# RTX1210 Rev.14.01.05 (Thu Nov  6 08:45:55 2014)
# MAC Address : ********
# Memory 256Mbytes, 3LAN, 1BRI
# main:  RTX1210 ver=00 serial=****** MAC-Address=*****
ess=***** MAC-Address=*****
# Reporting Date: Nov 3 16:04:53 2024
ip lan1 address 192.168.100.1/24
switch control use lan1 on terminal=on
switch control use lan2 on terminal=on
switch control use lan3 on terminal=on
ip filter 500000 restrict * * * * *
dhcp service server
dhcp server rfc2131 compliant except remain-silent
dhcp scope 1 192.168.100.2-192.168.100.191/24
dns host lan1
dns private address spoof on
schedule at 1 */Wed 00:00:00 * ntpdate ntp.nict.jp syslog
dashboard accumulate traffic on
```

### RTX1210のFWアップデート

:::message alert

[公式サイト](https://network.yamaha.com/support/rtx1210_boot/)にあるように、特定の製造番号のRTX1210は対策しないとFWアップデート後に起動しなくなる可能性があるので注意してください
>弊社製ギガアクセスVPNルーター「RTX1210」におきまして、ファームウェア更新後に起動しなくなるなどの不具合が発生することが判明いたしました。この不具合は、本記事に掲載する対策を施すことで回避することができます。

:::

工場出荷状態のFWだとIPv4 over IPv6のトンネリングの設定ができないのでFWをアップデートする必要があります。

※ IPoE方式だとルータの設定なしではIPv4の通信ができないのでIPv4 over IPv6の設定が必要です。


FWのアップデート方法は[こちらのPDF](https://www.rtpro.yamaha.co.jp/RT/manual/rtx1210/Users.pdf)を参考に行なってください。
筆者は外部メモリでアップデートしました。

### RTX1210の設定変更

RTX1210にアクセス後に`administrator`と入力すると設定を変更するモードに変わります。

```Shell
(RTX1210) > administrator 
Password: <パスワードを入力する（初期状態の場合は何も入力せずに「Enter」）>
(RTX1210) # 
```

#### 前提

- RTX1210の「LAN1」を家のネットワーク機器に接続します。
- RTX1210の「LAN2」をインターネットに接続します。
- 「VLANなし」の設定を記載します。
- インターネットプロバイダとの契約は「V6プラス」です。

#### IPv6の設定

**LAN2のDHCPv6クライアントの設定**
```Shell
ipv6 lan2 dhcp service client ir=on
```

LAN2がISPから「グローバルルーティングプレフィックス」を取得できるようになります。

その後の手順でLAN1のプレフィックスをLAN2で取得した「グローバルルーティングプレフィックス」に設定することで、LAN1内の機器がインターネットと通信できるようになります。

参考: [DHCPv6クライアント](https://wa3.i-3-i.info/word19691.html)

参考: [IPv6アドレス](https://www.infraexpert.com/study/ipv6z3.html)

**RAメッセージ送信の設定**
```Shell
ipv6 prefix 1 ra-prefix@lan2::/64
ipv6 lan1 rtadv send 1 o_flag=on
```

LAN1内の機器に対してルータがRA(Router Advertisement)メッセージを送信するようになります。

RAメッセージで配布するプレフィックスがLAN2で取得した「グローバルIPv6プレフィックス」になります。

参考: [Router Advertisementメッセージ](https://www.infraexpert.com/study/ipv6z12.html)

**LAN1のDHCPv6サーバの設定**
```Shell
ipv6 lan1 dhcp service server
```

LAN1内にDHCPv6サーバを構築します。

**LAN1のプレフィックスの設定**

```Shell
ipv6 lan1 address ra-prefix@lan2::1/64
```
LAN2のプレフィックスをLAN1に適用されます。

LAN1内のIPアドレスに「グローバルルーティングプレフィックス」が設定されるのでLAN1の機器はインターネットと通信ができるようになります。

**セキュリティ**
```shell
ipv6 lan2 secure filter in 200030 200031 200038 200039
ipv6 lan2 secure filter out 200099 dynamic 200080 200081 200082 200083 200084 200098 200099
ipv6 filter 200030 pass * * icmp6 * *
ipv6 filter 200031 pass * * 4
ipv6 filter 200038 pass * * udp * 546
ipv6 filter 200039 reject * *
ipv6 filter 200099 pass * * * * *
ipv6 filter dynamic 200080 * * ftp
ipv6 filter dynamic 200081 * * domain
ipv6 filter dynamic 200082 * * www
ipv6 filter dynamic 200083 * * smtp
ipv6 filter dynamic 200084 * * pop3
ipv6 filter dynamic 200098 * * tcp
ipv6 filter dynamic 200099 * * udp
```

#### IPv4の設定

IPv4 over IPv6トンネル
```Shell
tunnel select 1
 tunnel encapsulation map-e
 tunnel map-e type v6plus
 ip tunnel mtu 1460
 ip tunnel secure filter in 200030 200039
 ip tunnel secure filter out 200099 dynamic 200080 200082 200083 200084 200098 200099      
 ip tunnel nat descriptor 1000
 tunnel enable 1
```

IPv4 over IPv6については[こちらのHP](https://www.ntt.com/business/services/network/internet-connect/ocn-business/bocn/knowledge/archive_115.html)を参考にしてください。

※ IPoE方式だとルータの設定なしではIPv4の通信ができないのでIPv4 over IPv6の設定が必要です。

**セキュリティ**
```Shell
ip filter 200030 pass * 192.168.100.0/24 icmp * *
ip filter 200039 reject * *
ip filter 200099 pass * * * * *
ip filter dynamic 200080 * * ftp
ip filter dynamic 200082 * * www
ip filter dynamic 200083 * * smtp
ip filter dynamic 200084 * * pop3
ip filter dynamic 200098 * * tcp
ip filter dynamic 200099 * * udp
```

**NAT**
```shell
nat descriptor type 1000 masquerade
nat descriptor address outer 1000 map-e
```

**DNS**
```shell
dns host lan1
dns service fallback on
dns server dhcp lan2
```

**DHCP**
```shell
dhcp service server
dhcp server rfc2131 compliant except remain-silent
dhcp scope 1 192.168.100.2-192.168.100.191/24
```

**デフォルトデートウェイ**
```shell
ip route default gateway tunnel 1
```

#### その他の設定

ポート8に繋いだPCでその他のポートの通信をみたいので、ポートミラーリングの設定をします。
```Shell
lan port-mirroring lan1 8 in 1 2 3 4 5 6 7 out 1 2 3 4 5 6 7
```

LAN1のポート1~7の受信・送信された通信データがポート8にコピーされます。

:::details 全体設定

```Shell
# RTX1210 Rev.14.01.42 (Fri Jul  5 11:17:45 2024)
# MAC Address : ******, ******, ******
# Memory 256Mbytes, 3LAN, 1BRI
# main:  RTX1210 ver=00 serial=****** MAC-Address=****** MAC-Addr
ess=****** MAC-Address=****** TAM=11
# Reporting Date: Dec 7 15:38:26 2024
console character ja.utf8
ip route default gateway tunnel 1
ipv6 prefix 1 ra-prefix@lan2::/64
lan port-mirroring lan1 8 in 1 2 3 4 5 6 7 out 1 2 3 4 5 6 7
ip lan1 address 192.168.100.1/24
ipv6 lan1 address ra-prefix@lan2::1/64
ipv6 lan1 rtadv send 1 o_flag=on
ipv6 lan1 dhcp service server
switch control use lan1 on terminal=on
ipv6 lan2 secure filter in 200030 200031 200038 200039
ipv6 lan2 secure filter out 200099 dynamic 200080 200081 200082 200083 200084 200098 200099
ipv6 lan2 dhcp service client ir=on
ngn type lan2 ntt
tunnel select 1
 tunnel encapsulation map-e
 tunnel map-e type v6plus
 ip tunnel mtu 1460
 ip tunnel secure filter in 200030 200039
 ip tunnel secure filter out 200099 dynamic 200080 200082 200083 200084 200098 200099      
 ip tunnel nat descriptor 1000
 tunnel enable 1
ip filter 200030 pass * 192.168.100.0/24 icmp * *
ip filter 200039 reject * *
ip filter 200099 pass * * * * *
ip filter dynamic 200080 * * ftp
ip filter dynamic 200082 * * www
ip filter dynamic 200083 * * smtp
ip filter dynamic 200084 * * pop3
ip filter dynamic 200098 * * tcp
ip filter dynamic 200099 * * udp
nat descriptor type 1000 masquerade
nat descriptor address outer 1000 map-e
ipv6 filter 200030 pass * * icmp6 * *
ipv6 filter 200031 pass * * 4
ipv6 filter 200038 pass * * udp * 546
ipv6 filter 200039 reject * *
ipv6 filter 200099 pass * * * * *
ipv6 filter dynamic 200080 * * ftp
ipv6 filter dynamic 200081 * * domain
ipv6 filter dynamic 200082 * * www
ipv6 filter dynamic 200083 * * smtp
ipv6 filter dynamic 200084 * * pop3
ipv6 filter dynamic 200098 * * tcp
ipv6 filter dynamic 200099 * * udp
dhcp service server
dhcp server rfc2131 compliant except remain-silent
dhcp scope 1 192.168.100.2-192.168.100.191/24
dns host lan1
dns service fallback on
dns server dhcp lan2
```

:::

### RTX1210とインターネットの接続

RTX1210のLAN2にインターネットとのLANケーブルを接続します。

### 家のネットワークの無線対応

今のままではRTX1210と繋がっている機器しかインターネットと通信できないです。

無線LANルータとRTX1210を接続することでAmazon EchoなどのWi-fi対応機器をインターネットと通信できるようにします。

RTX1210のLAN1のポート1~7の任意のポートと無線LANルータを接続します。

:::message
無線でインターネットと通信できない時は、割り振られているIPアドレスが正しいか確認してください。

無線LANルータのDHCPで割り当てたIPアドレスになってしまっているなど
:::

## 通信の確認

**通信確認用PCにWiresharkをインストール**

[Wireshark](https://www.wireshark.org/download.html)

自分はMacにインストールしました。

**通信確認用PCとRTX1210の接続**

Mac(USBタイプのNIC)とRTX1210のLAN1のポート8をLANケーブルで接続します。

※ ポートミラーリングしたポートを接続してください。

**Wiresharkで通信監視**

USB NIC(筆者の環境ではAX88179A)を選択してWiresharkを起動します。
![](/images/homenetwork-ntopng/wireshark-usb.png)
*Wiresharkの対象選択画面*

![](/images/homenetwork-ntopng/wireshark-switch.png)
*Wiresharkの通信監視画面*

![](/images/homenetwork-ntopng/switch-net.jpg)
*ゲーム機のIPアドレス*

記事の冒頭で見たようにゲーム機など家のネットワーク通信が確認できました。

## まとめ

ポートミラーリング + Wiresharkで家のネットワーク通信を見ることができました。

次記事書くなら、下記あたりかなーの気持ちです。
- ntopng導入
- IPv6あたり
- TURNやSTUNサーバについて

## アドベントカレンダー予告

明日は [yuta_tsuge](https://qiita.com/yuta_tsuge) さんが投稿します！！！

「[レバテック開発部 Advent Calendar 2024](https://qiita.com/advent-calendar/2024/levtech)」をぜひご購読ください🥳