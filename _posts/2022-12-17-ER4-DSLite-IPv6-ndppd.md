---
layout: post
title: "EdgeRouter4 で DS-Lite(transix) + IPv6(ひかり電話なし, ndppd)"
categories: Network
---
- [構成図](#構成図)
- [設定](#設定)
  - [1. ER-4に接続する](#1-er-4に接続する)
  - [2. WAN から割り当てられているIPv6アドレスを取得する](#2-wan-から割り当てられているipv6アドレスを取得する)
  - [3. DS-Lite の設定](#3-ds-lite-の設定)
  - [4. DHCP, DNS の設定](#4-dhcp-dns-の設定)
  - [5. ND Proxy (ndppd) の設定](#5-nd-proxy-ndppd-の設定)
    - [ndppd を MIPS64 向けにビルドする](#ndppd-を-mips64-向けにビルドする)
    - [ndppd の設定](#ndppd-の設定)
  - [6. IPv6 RA の設定](#6-ipv6-ra-の設定)
  - [6. 確認](#6-確認)
- [参考](#参考)


## 構成図
![](https://user-images.githubusercontent.com/33781977/208204660-97d3e191-64f3-44c2-8d02-54e9f40613a5.png)

## 設定
* ここでは タイムゾーン, NTP, ファイアウォールなどの設定は省きます

### 1. ER-4に接続する
1. CONSOLE にコンソールケーブルを繋ぐ
2. PuTTY などからシリアル接続、以下設定
    * Serial line
        * COM3 (COM番号はデバイスマネージャー -> ポート(COMとLPT)から確認)
    * Speed
        * 115200
    * Connection type
        * Serial
3. 接続出来たら初期パスワードでログイン
    * id
        * ubnt
    * password
        * ubnt

### 2. WAN から割り当てられているIPv6アドレスを取得する
IPv6 を自動で割り当てられるように設定する
```bash
configure
set interfaces ethernet eth0 ipv6 address autoconf
commit
exit
```

10分程度待つと IPv6 アドレスが降ってくるので、このアドレスをメモしておく
```bash
show interfaces
Codes: S - State, L - Link, u - Up, D - Down, A - Admin Down
Interface    IP Address                        S/L  Description
---------    ----------                        ---  -----------
eth0         192.168.1.1/24                    u/u
             XXXX:XXXX:XXXX:XXXX:YYYY:YYYY:YYYY:YYYY/64
eth1         -                                 u/D
eth2         -                                 u/D
eth3         -                                 u/D
...
```
next hop に設定する対向ルータのリンクローカルアドレスもメモしておく (`fe80::` から始まるアドレス)
```bash
show ipv6 neighbors | grep eth0
fe80::21f:6cff:YYYY:YYYY dev eth0 lladdr 00:1f:6c:27:12:c5 router REACHABLE
```

### 3. DS-Lite の設定
IPv6 が割り当てられた状態では静的割当でエラーになったりするので消しておく
```bash
sudo ip addr flush dev eth0
```

IPIP トンネルの設定
```bash
configure
delete interfaces ethernet eth0 ipv6
# 割り当てられた IPv6 アドレス
set interfaces ethernet eth0 address XXXX:XXXX:XXXX:XXXX:YYYY:YYYY:YYYY:YYYY/64
set interfaces ipv6-tunnel v6tun0 encapsulation ipip6
# 割り当てられた IPv6 アドレス (/64 なし)
set interfaces ipv6-tunnel v6tun0 local-ip XXXX:XXXX:XXXX:XXXX:YYYY:YYYY:YYYY:YYYY
# transix の AFTR
set interfaces ipv6-tunnel v6tun0 remote-ip 2404:8e00::feed:102
set interfaces ipv6-tunnel v6tun0 mtu 1500
set interfaces ipv6-tunnel v6tun0 multicast disable
set protocols static interface-route 0.0.0.0/0 next-hop-interface v6tun0
# 対向ルータのリンクローカルアドレス
set protocols static route6 ::/0 next-hop fe80::21f:6cff:YYYY:YYYY interface eth0
set interfaces ipv6-tunnel v6tun0 ttl 64
commit; save
exit
```

ここまでで IPv4 での `ping` が通るようになる
```
ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=60 time=5.05 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=60 time=3.43 ms
...
```

### 4. DHCP, DNS の設定
```
configure
set system name-server 2001:4860:4860::8888
set system name-server 2001:4860:4860::8844
set system name-server 8.8.8.8
set system name-server 8.8.4.4

set service dhcp-server disabled false
set service dhcp-server hostfile-update disable
set service dhcp-server shared-network-name LAN1 authoritative enable
set service dhcp-server shared-network-name LAN1 subnet 192.168.1.0/24
set service dhcp-server shared-network-name LAN1 subnet 192.168.1.0/24 default-router 192.168.1.1
set service dhcp-server shared-network-name LAN1 subnet 192.168.1.0/24 dns-server 8.8.8.8
set service dhcp-server shared-network-name LAN1 subnet 192.168.1.0/24 dns-server 8.8.4.4

delete interfaces ethernet eth0 address
set interfaces ethernet eth0 address dhcp

commit; save
exit
```

### 5. ND Proxy (ndppd) の設定
以下を参考に行いました。本当に情報が少なく、正直この記事がなければ出来なかったので感謝します。
* [EdgeRouter 4/6でNuro光を使う](https://www.blog.slow-fire.net/2021/04/04/edgerouter-4-6%E3%81%A7nuro%E5%85%89%E3%82%92%E4%BD%BF%E3%81%86/)
* [ムカつくw Edgerouter X、Edgerouter 4 / 6の情報がない。　クロスコンパイラを使う](https://www.blog.slow-fire.net/2021/04/04/%e3%83%a0%e3%82%ab%e3%81%a4%e3%81%8fw-edgerouter-x%e3%80%81edgerouter-4-6%e3%81%ae%e6%83%85%e5%a0%b1%e3%81%8c%e3%81%aa%e3%81%84%e3%80%82%e3%80%80%e3%82%af%e3%83%ad%e3%82%b9%e3%82%b3%e3%83%b3/)


#### ndppd を MIPS64 向けにビルドする
```bash
# amd64 な環境を用意する
docker run -it amd64/ubuntu bash

# make に必要なものを揃える
apt -y install g++-mips64-linux-gnuabi64 vim git make

# ndppd を落としてビルド
git clone -b master https://github.com/DanielAdolfsson/ndppd.git
cd ndppd
cp Makefile Makefile-edge
vim Makefile-edge
# Makefile を以下のように設定する
"""
PREFIX = /config/ndppd/local
CXX = /usr/bin/mips64-linux-gnuabi64-g++
LDFLAGS = -static
"""

# make
make -f Makefile-edge
```

ホストに戻って出来上がった ndppd を取ってくる
```bash
docker ps
CONTAINER ID   IMAGE          COMMAND   CREATED      STATUS              PORTS     NAMES
e1667963bed6   amd64/ubuntu   "bash"    4 days ago   Up About a minute             happy_morse

docker cp happy_morse:/ndppd/ndppd .

```

`file` を見て以下のようになっていればok
```bash
file ndppd
ndppd: ELF 64-bit MSB executable, MIPS, MIPS64 rel2 version 1 (GNU/Linux), statically linked, BuildID[sha1]=593de2ff76b5d02db60642f450af9cac728389c5, for GNU/Linux 3.2.0, not stripped
```

ER-4 へ ndppd を転送する
```bash
sftp ubnt@192.168.1.1
> put ndppd /
```

#### ndppd の設定
ndppd を配置し実行権限を与える
```bash
mv /ndppd /config/ndppd/local/sbin/ndppd
chmod +x /config/ndppd/local/sbin/ndppd
```

ndppd.conf  
rule には割り当てられた IPv6 のインターフェースID部を0にしたものを設定する
```
cat << 'EOF' > /config/ndppd/ndppd.conf
proxy eth0 {
    router no
    timeout 500
    autowire yes
    keepalive yes
    retries 3
    ttl 30000
    rule XXXX:XXXX:XXXX:XXXX::/64 {
        iface eth1
    }
}

proxy eth1 {
    router yes
    timeout 500
    autowire yes
    keepalive yes
    retries 3
    ttl 30000
    rule XXXX:XXXX:XXXX:XXXX::/64 {
        auto
    }
}
EOF
```

ER-4 起動時に ndppd が実行されるように `post-config.d` に起動用スクリプトを作成する
```bash
cat << 'EOF' > /config/scripts/post-config.d/ndppd.sh
#!/bin/vbash
# copy to this file into
# /config/scripts/post-config.d/
/config/ndppd/ndppd.initscript start
EOF

cat << 'EOF' > /config/ndppd/ndppd.initscript
#!/bin/sh

# kFreeBSD do not accept scripts as interpreters, using #!/bin/sh and sourcing.
if [ true != "$INIT_D_SCRIPT_SOURCED" ] ; then
set "$0" "$@"; INIT_D_SCRIPT_SOURCED=true . /lib/init/init-d-script
fi

DESC="NDP Proxy Daemon"
PIDFILE=/run/ndppd.pid
CONFIGFILE=/config/ndppd/ndppd.conf
DAEMON=/config/ndppd/local/sbin/ndppd
DAEMON_ARGS="-d -c $CONFIGFILE -p $PIDFILE"
EOF

chmod +x /config/scripts/post-config.d/ndppd.sh
chmod +x /config/ndppd/ndppd.initscript
```

ndppd を起動
```bash
/config/scripts/post-config.d/ndppd.sh

# 起動確認
ps ax | grep ndppd
 3130 ?        Ss     0:20 /config/ndppd/local/sbin/ndppd -d -c /config/ndppd/ndppd.conf -p /run/ndppd.pid
 6690 pts/0    S+     0:00 grep ndppd
```

### 6. IPv6 RA の設定
```bash
configure
set interfaces ethernet eth1 ipv6 address autoconf
set interfaces ethernet eth1 ipv6 address eui64 XXXX:XXXX:XXXX:XXXX::/64
set interfaces ethernet eth1 ipv6 dup-addr-detect-transmits 1
set interfaces ethernet eth1 ipv6 router-advert cur-hop-limit 64
set interfaces ethernet eth1 ipv6 router-advert link-mtu 1500
set interfaces ethernet eth1 ipv6 router-advert managed-flag false
set interfaces ethernet eth1 ipv6 router-advert max-interval 600
set interfaces ethernet eth1 ipv6 router-advert other-config-flag true
set interfaces ethernet eth1 ipv6 router-advert prefix ::/64
set interfaces ethernet eth1 ipv6 router-advert prefix ::/64 autonomous-flag true
set interfaces ethernet eth1 ipv6 router-advert prefix ::/64 on-link-flag true
set interfaces ethernet eth1 ipv6 router-advert prefix ::/64 valid-lifetime 2592000
set interfaces ethernet eth1 ipv6 router-advert reachable-time 0
set interfaces ethernet eth1 ipv6 router-advert retrans-timer 0
set interfaces ethernet eth1 ipv6 router-advert send-advert true
commit; save
```

### 6. 確認
ここまでの設定で `eth1` に接続された端末で IPv6 が使えるようになっているはずなので、[ipv6-test.com](ipv6-test.com) などで確認。


## 参考
* [EdgeRouter設定メモ: IPv6/IPoE + DS-Liteでインターネット高速化](https://stop-the-world.hatenablog.com/entry/2018/11/05/135911)
* [EdgeRouter 4/6でNuro光を使う](https://www.blog.slow-fire.net/2021/04/04/edgerouter-4-6%E3%81%A7nuro%E5%85%89%E3%82%92%E4%BD%BF%E3%81%86/)
* [ムカつくw Edgerouter X、Edgerouter 4 / 6の情報がない。　クロスコンパイラを使う](https://www.blog.slow-fire.net/2021/04/04/%e3%83%a0%e3%82%ab%e3%81%a4%e3%81%8fw-edgerouter-x%e3%80%81edgerouter-4-6%e3%81%ae%e6%83%85%e5%a0%b1%e3%81%8c%e3%81%aa%e3%81%84%e3%80%82%e3%80%80%e3%82%af%e3%83%ad%e3%82%b9%e3%82%b3%e3%83%b3/)
* [Edgerouter X(ER-X) 用に ndppd をカンタンにビルドする (ひかり電話なし)](https://qiita.com/sho7650/items/da9bc34f1640223aaf99)

