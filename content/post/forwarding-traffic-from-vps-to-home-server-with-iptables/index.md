---
title: "iptablesでVPSから自宅サーバにトラフィックを流す"
slug: forwarding-traffic-from-vps-to-home-server-with-iptables
author: ""
type: ""
date: 2022-11-27T09:42:30+09:00
draft: false
subtitle: ""
image: ""
toc: true
tocMove: true
tags: ["iptables", "Tailscale", "Docker Swarm"]
---

自宅サーバとVPSでSwarmクラスタを構成し、VPSでインターネットからのアクセスを受けることで、自宅ルータのポートを開放せずにセルフホストしたサービスに自宅内外からアクセスできるようにしていた。

- [TailscaleネットワークでSwarmクラスタを構築 - blog.bridgey.dev]({{< ref "post/docker-swarm-with-tailscale/index.md" >}})
- [Cloudflare、Traefik、Authentikを使った自宅Dockerサーバ - blog.bridgey.dev]({{< ref "post/docker-home-server-with-cloudflare-traefik-and-authentik/index.md" >}})

なぜ過去形かというと、11月に入ってからSwarmに乗せたコンテナ間でうまく通信できなくなってしまったためだ。

そのため、現在はSwarmではなく、自宅サーバ上でComposeを使ってサービスを立てている。

## VPSから自宅サーバへトラフィックを流すための設定

これまでVPSに届いたトラフィックを自宅サーバに流してくれていた、Swarmのルーティングメッシュの代わりを用意する必要がある。

VPSと自宅サーバへの経路はTailscaleで確保できているため、以下のスクリプトのようにVPSのiptablesでトラフィックを転送するようにした。

```sh
#!/bin/sh
INTERNAL_SERVER=192.0.2.1  # 実際は自宅サーバのTailscaleのIPアドレス

iptables -t nat -X wormhole-prerouting

# PREROUTING
iptables -t nat -N wormhole-prerouting
iptables -t nat -A wormhole-prerouting -j MARK --set-mark 0x40000/0xff0000
iptables -t nat -A wormhole-prerouting -i eth0 -p tcp -m tcp --dport 80 -j DNAT --to-destination ${INTERNAL_SERVER}:80
iptables -t nat -A wormhole-prerouting -i eth0 -p tcp -m tcp --dport 443 -j DNAT --to-destination ${INTERNAL_SERVER}:443
iptables -t nat -I PREROUTING -i eth0 -p tcp -m multiport --dports 80,443 -j wormhole-prerouting
```

TailscaleがFORWARDするパケットの識別に使っているMARKを活用している。

ちなみに、VPS上でsocatコンテナを立てる方法でも同じ効果を得られることを確認している。

```yaml
version: '3.9'

services:
  http-redirect:
    image: alpine/socat
    init: true
    ports:
      - 80:80
    command: -d -d tcp-listen:80,fork tcp-connect:192.0.2.1:80

  https-redirect:
    image: alpine/socat
    init: true
    ports:
      - 443:443
    command: -d -d tcp-listen:443,fork tcp-connect:192.0.2.1:443
```

## 余談: 原因候補

上記問題が発生したのが体調を崩して寝込んでいた期間であったため、原因は特定できていない。

ちょうどその期間中、あるいは直前に、自宅サーバのRAIDを構成しているHDDの1つが壊れたり、Swarmクラスタの実現に利用していたTailscaleをアップデートがあったりしたように思う。

Tailscaleを使わずにSwarmクラスタを作ってみればよいのだが、面倒なのでサボっている。

## おわりに

直近は体調崩したうえに、マシンも不調になるという泣き面に蜂状態であった。

タイトルとは関係ないが、別用途で使っていたラズパイも同時期に壊れている。

代わりのラズパイ買わないと。
