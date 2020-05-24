---
title: "EdgeRouter XのIPv6対応（ひかり電話なし）"
author: ""
type: ""
date: 2020-05-23T21:20:12+09:00
draft: false
subtitle: ""
image: ""
toc: true
tocMove: true
tags: ["ルータ", "EdgeRouter", "IPv6"]
---

[EdgeRourer Xを買ってから]({{<ref "/post/edge-router/index.md">}})、ずっとやろうと思ってやっていませんでしたが、ようやくIPv6を使用するための設定をしました。  

{{<tweet 1256981117141266432>}}

(設定してから記事にするまでにも間があいてしまいました)

備忘録としてやったことをまとめます。  
本記事には試行錯誤の過程をすべて記載しております。  
記載されている設定は、IPv6を使えるようにする最小限の設定ではない点にご注意ください。

## IPv6のファイアウォール設定
参考ページ: [EdgeRouterでIPoE(IPv6)インターネットを接続を行う(ひかり電話あり) – nosense](http://www.nosense.jp/edgerouter-ipoe/)

EdgeRouterにログインし、下記のコマンドを実行する。  

```
configure
set firewall ipv6-name WANv6_IN default-action drop
set firewall ipv6-name WANv6_IN description 'WAN inbound traffic forwarded to LAN'
set firewall ipv6-name WANv6_IN enable-default-log
set firewall ipv6-name WANv6_IN rule 10 action accept
set firewall ipv6-name WANv6_IN rule 10 description 'Allow established/related sessions'
set firewall ipv6-name WANv6_IN rule 10 state established enable
set firewall ipv6-name WANv6_IN rule 10 state related enable
set firewall ipv6-name WANv6_IN rule 20 action drop
set firewall ipv6-name WANv6_IN rule 20 description 'Drop invalid state'
set firewall ipv6-name WANv6_IN rule 20 state invalid enable
set firewall ipv6-name WANv6_LOCAL default-action drop
set firewall ipv6-name WANv6_LOCAL description 'WAN inbound traffic to the router'
set firewall ipv6-name WANv6_LOCAL enable-default-log
set firewall ipv6-name WANv6_LOCAL rule 10 action accept
set firewall ipv6-name WANv6_LOCAL rule 10 description 'Allow established/related sessions'
set firewall ipv6-name WANv6_LOCAL rule 10 state established enable
set firewall ipv6-name WANv6_LOCAL rule 10 state related enable
set firewall ipv6-name WANv6_LOCAL rule 20 action drop
set firewall ipv6-name WANv6_LOCAL rule 20 description 'Drop invalid state'
set firewall ipv6-name WANv6_LOCAL rule 20 state invalid enable
set firewall ipv6-name WANv6_LOCAL rule 30 action accept
set firewall ipv6-name WANv6_LOCAL rule 30 description 'Allow IPv6 icmp'
set firewall ipv6-name WANv6_LOCAL rule 30 protocol ipv6-icmp
set firewall ipv6-name WANv6_LOCAL rule 40 action accept
set firewall ipv6-name WANv6_LOCAL rule 40 description 'allow dhcpv6'
set firewall ipv6-name WANv6_LOCAL rule 40 destination port 546
set firewall ipv6-name WANv6_LOCAL rule 40 protocol udp
commit
exit
```

## pppdの置き換え

IPv6 PPPoEが正しく動くようにする。  
参考ページ: [EdgeRouterでIPv6 PPPoEが正しく接続できない問題を解決する - もやし日誌](https://kazblog.hateblo.jp/entry/2018/08/06/185940)

`/sbin/pppd`をパッチをあてたものに置き換える。  

- [ここ](https://github.com/tumugin/edgeos-ppp/releases)から`ppp_mipsel.tar.gz`をダウンロードし、展開する  

  ```
  tar xf ppp_mipsel.tar.gz
  ```

- 展開されたファイルをEdgeRouterに転送する (\<xxx\>の部分は適切な値に置き換える)  

  ```
  scp -rC 2.4.7/ pppd <username>@<EdgeRouter's IP address>:/home/<username>/
  ```

- EdgeRouterにログインし、転送したファイルを設置する  
  ```
  sudo su -
  cp /sbin/pppd /sbin/pppd_old
  cp /home/<username>/pppd /sbin/
  cp -r /home/<username>/2.4.7 /usr/lib/pppd/
  cp /etc/ppp/ip-pre-up /etc/ppp/ipv6-pre-up
  cp /etc/ppp/ip-up.d/* /etc/ppp/ipv6-up.d/
  ```

- EdgeRouterを再起動  


ファームウェアを更新すると置き換えたファイルは消えてしまうため、上記手順を再度実行する。  

## EdgeRouterにIPv6アドレスを自動で割り当てるための設定

参考ページ: [EdgeRouter設定メモ: IPv6/IPoE + DS-Liteでインターネット高速化 - stop-the-world](http://stop-the-world.hatenablog.com/entry/2018/11/05/135911)

EdgeRouterにログインし、下記のコマンドを実行する。  
```
configure
set interfaces ethernet eth0 ipv6 address autoconf
commit
exit
```

## ndppdの配置

ひかり電話がない環境でもLAN内にIPv6アドレスを割り当てるために、ndppdを配置する。  

### ndppdのクロスコンパイル
参考ページ: [EdgeRouter X (ER-X) で ndppd を動かす - Qiita](https://qiita.com/ikai/items/b1b410a62aa68d1a0eec)

EdgeRouterで動作するように、Dockerを用いてクロスコンパイルする。  
```
git clone https://github.com/DanielAdolfsson/ndppd.git
docker run -v $PWD/ndppd:/ndppd -it --rm ubuntu  # 本コマンドでコンテナ内に入るので、以降のコマンドはコンテナ内で実行する
apt-get update -y
apt-get install -y g++-mipsel-linux-gnu make vim  # vimはMakefileを修正するためにインストールした
cd /ndppd/
vim Makefile  # 修正内容は下記"Makefileの修正内容"を参照
make
```

#### Makefileの修正内容

```
$ git diff Makefile
diff --git a/Makefile b/Makefile
index 00ca22b..f044009 100644
--- a/Makefile
+++ b/Makefile
@@ -1,11 +1,11 @@
 ifdef DEBUG
 CXXFLAGS ?= -g -DDEBUG
 else
-CXXFLAGS ?= -O3
+CXXFLAGS ?= -O3 -static
 endif

-PREFIX  ?= /usr/local
-CXX     ?= g++
+PREFIX  ?= /ndppd/local
+CXX     ?= /usr/bin/mipsel-linux-gnu-g++
 GZIP    ?= /bin/gzip
 MANDIR  ?= ${DESTDIR}${PREFIX}/share/man
 SBINDIR ?= ${DESTDIR}${PREFIX}/sbin
@@ -37,13 +37,13 @@ ndppd.conf.5.gz:
        ${GZIP} < ndppd.conf.5 > ndppd.conf.5.gz

 ndppd: ${OBJS}
-       ${CXX} -o ndppd ${LDFLAGS} ${OBJS} ${LIBS}
+       /usr/bin/mipsel-linux-gnu-g++ -o ndppd -static ${LDFLAGS} ${OBJS} ${LIBS}

 nd-proxy: nd-proxy.c
-       ${CXX} -o nd-proxy -Wall -Werror ${LDFLAGS} `${PKG_CONFIG} --cflags glib-2.0` nd-proxy.c `${PKG_CONFIG} --libs glib-2.0`
+        /usr/bin/mipsel-linux-gnu-g++ -o nd-proxy -Wall -Werror ${LDFLAGS} `${PKG_CONFIG} --cflags glib-2.0` nd-proxy.c `${PKG_CONFIG} --libs glib-2.0`

 .cc.o:
-       ${CXX} -c ${CPPFLAGS} $(CXXFLAGS) -o $@ $<
+       /usr/bin/mipsel-linux-gnu-g++ -c ${CPPFLAGS} $(CXXFLAGS) -o $@ $<

 clean:
        rm -f ndppd ndppd.conf.5.gz ndppd.1.gz ${OBJS} nd-proxy
```

### EdgeRouterに設置
コンパイルしたndppdをEdgeRouterに転送し、配置する。  

- EdgeRouterへコンパイルしたファイルを転送する  
  ```
  scp -C ndppd <username>@<EdgeRouter's IP address>:/home/<username>/
  ```

- EdgeRouterにログインし、転送したファイルを設置する  
  ```
  sudo su -
  cp /home/<username>/ndppd /config/user-data/ndppd/
  chmod 755 /config/user-data/ndppd/ndppd
  ```

### ndppdの設定

ndppdの設定ファイルを設置する。  
ndppdの起動スクリプトで設定ファイルを指定しない場合のファイルパスは、`/etc/ndppd.conf`

```
sudo su -
vi /config/user-data/ndppd/ndppd.conf  # 下記の内容を記述
```

#### ndppd.conf

```
proxy eth0 {
   router no
   timeout 500
   autowire yes
   keepalive yes
   retries 3
   ttl 30000
   rule ::/0 {
      iface switch0
   }
}

proxy switch0 {
   router yes
   timeout 500
   autowire yes
   keepalive yes
   retries 3
   ttl 30000
   rule ::/0 {
      auto
   }
}
```

## 自動起動設定

ndppdが自動で起動するように、2つのスクリプトを設置する。  
参考ページ: [Set up IPv6 without Prefix Delegation | Ubiquiti Community](https://community.ui.com/questions/Set-up-IPv6-without-Prefix-Delegation/ea0998be-2d13-4747-ac1e-9a26cb1a2976)

- デーモン起動用スクリプト  
  ```
  sudo su -
  vi /config/user-data/ndppd/ndppd.initscript  # 下記の内容を記述
  chmod 755 ndppd.initscript
  ```
  **/config/user-data/ndppd/ndppd.initscript**
  ```
  #!/bin/bash
  ACTION=$1
  NDPPD_DIR="/config/user-data/ndppd"
  NDPPD_BIN="$NDPPD_DIR/ndppd"
  NDPPD_CONF="$NDPPD_DIR/ndppd.conf"
  NDPPD_PID="/run/ndppd.pid"

  do_stop(){
          if [ -f $NDPPD_PID ]; then
                  kill `cat $NDPPD_PID` > /dev/null 2>&1
                  rm -f $NDPPD_PID
          fi
          echo 0 > /proc/sys/net/ipv6/conf/all/proxy_ndp
          echo 0 > /proc/sys/net/ipv6/conf/default/proxy_ndp
          /sbin/ifconfig switch0 -promisc
          /sbin/ifconfig eth4 -promisc
          /sbin/ifconfig eth3 -promisc
          /sbin/ifconfig eth2 -promisc
          /sbin/ifconfig eth1 -promisc
          /sbin/ifconfig eth0 -promisc
  }
  do_start(){
          if [ -f $NDPPD_CONF ]; then
                  echo "Starting NDP Proxy Daemon ..."
                  /sbin/ifconfig eth0 promisc
                  /sbin/ifconfig eth1 promisc
                  /sbin/ifconfig eth2 promisc
                  /sbin/ifconfig eth3 promisc
                  /sbin/ifconfig eth4 promisc
                  /sbin/ifconfig switch0 promisc
                  echo 1 > /proc/sys/net/ipv6/conf/default/proxy_ndp
                  echo 1 > /proc/sys/net/ipv6/conf/all/proxy_ndp
                  $NDPPD_BIN -d -c $NDPPD_CONF -p $NDPPD_PID > /dev/null 2>&1
          else
                  echo "ERROR: $NDPPD_CONF not found"
                  exit 1
          fi
  }
  case "$ACTION" in
          start)
                  do_start
          ;;
          restart)
                  do_stop
                  do_start
          ;;
          stop)
                  do_stop
          ;;
          *)
                  echo "$1 start|stop|restart"
          ;;
  esac
  exit 0
  ```

- 自動起動用スクリプト  
  ```
  sudo su -
  vi /config/scripts/post-config.d/ndppd  # 下記の内容を記述
  chmod 755 /config/scripts/post-config.d/ndppd
  ```
  **/config/scripts/post-config.d/ndppd**
  ```
  #!/bin/bash
  /config/user-data/ndppd/ndppd.initscript start
  ```

## LAN内の機器にIPv6アドレスを割り当てるための設定
参考ページ: [EdgeRouter X (ER-X) で IPv6 IPoE + DS-Lite するまで - Qiita](https://qiita.com/ikai/items/292050d8fa2f185e654a)

- ndppdを起動しておく  
[自動起動設定](#自動起動設定)後に再起動していない場合は、ndppdを起動する。  
  ```
  sudo /config/scripts/post-config.d/ndppd
  ```

- EdgeRouterのWeb UIから、割り振られたIPv6アドレスを確認しておく
- EdgeRouterにログインし、下記を実行する  
2行目のシングルクォート内の値は`xxxx:xxxx:xxxx:xxxx::64`のような文字列になります。  
ちなみに、上4桁ではなく、割り振られたIPv6アドレスの末尾に`/64`を付けて設定した場合でも、LAN内の機器にIPv6アドレスが割り振られました。  
  ```
  configure
  set interfaces switch switch0 ipv6 address eui64 '<EdgeRouterに割り振られたIPv6の上4桁(:区切り)>::/64'
  set interfaces switch switch0 ipv6 dup-addr-detect-transmits 1
  set interfaces switch switch0 ipv6 router-advert cur-hop-limit 64
  set interfaces switch switch0 ipv6 router-advert link-mtu 1500
  set interfaces switch switch0 ipv6 router-advert managed-flag false
  set interfaces switch switch0 ipv6 router-advert max-interval 600
  set interfaces switch switch0 ipv6 router-advert other-config-flag true
  set interfaces switch switch0 ipv6 router-advert prefix '::/64' autonomous-flag true
  set interfaces switch switch0 ipv6 router-advert prefix '::/64' on-link-flag true
  set interfaces switch switch0 ipv6 router-advert prefix '::/64' valid-lifetime 2592000
  set interfaces switch switch0 ipv6 router-advert reachable-time 0
  set interfaces switch switch0 ipv6 router-advert retrans-timer 0
  set interfaces switch switch0 ipv6 router-advert send-advert true
  commit
  exit
  ```
