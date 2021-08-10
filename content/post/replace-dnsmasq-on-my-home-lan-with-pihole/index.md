---
title: "自宅ネットワークでDnsmasqが担っていた機能をPi-Holeに移行した"
slug: replace-dnsmasq-on-my-home-lan-with-pihole
author: ""
type: ""
date: 2021-08-10T00:05:03+09:00
draft: false
subtitle: ""
image: ""
toc: false
tocMove: false
tags: ['Dnsmasq', 'Pi-Hole', 'Docker']
---

自宅ネットワークにDnsmasqを設置し、以下の役割を持たせていた。

- DNSキャッシュサーバ
- DHCPサーバ
- 広告ブロック

DnsmasqはラズパイのOS上で動作させていたので、環境移行に備えてDockerコンテナ化することにした。
その際、上記機能の実現に広告ブロック機能を備えたDNSサーバであるPi-Holeを使ってみた。

[Pi-hole – Network-wide protection](https://pi-hole.net/)

Pi-Holeは内部にDnsmasqを持っており、DHCPサーバとして使うことができる。
広告リストのアップデート機能は組み込まれているため、自前でスクリプトを書かずに済む（楽ができる）という点も、Pi-Holeを使うことにした理由の一つである。

各設定ファイルは以下の通り。

## `docker-compose.yml`  

```yaml
version: "3"

services:
  pihole:
    container_name: pihole
    image: testdasi/pihole-dot-doh:latest
    network_mode: host
    environment:
      TZ: Asia/Tokyo
      PIHOLE_DNS_: '127.2.2.2#5253;0::1#5253'
      DNSMASQ_LISTENING: all
    env_file:
      - ./pihole.env
    # Volumes store your data between container upgrades
    volumes:
       - ./etc-pihole/:/etc/pihole/
       - ./etc-dnsmasq.d/:/etc/dnsmasq.d/
       - ./stubby.yml:/config/stubby.yml
    # Recommended but not required (DHCP needs NET_ADMIN)
    #   https://github.com/pi-hole/docker-pi-hole#note-on-capabilities
    cap_add:
      - NET_ADMIN
    restart: unless-stopped
```
私は[DoTを有効にしたイメージ](https://github.com/testdasi/pihole-dot-doh)を使っている。  
ベースは公式イメージなので、広告リストのアップデート機能を備えている[^1]。  

[^1]: [pi-hole/docker-pi-hole: Pi-hole in a docker container](https://github.com/pi-hole/docker-pi-hole)

{{< blockquote author="pi-hole/docker-pi-hole: Pi-hole in a docker container" link="https://github.com/pi-hole/docker-pi-hole">}}
Automatic Ad List Updates - since the 3.0+ release, cron is baked into the container and will grab the newest versions of your lists and flush your logs. Set your TZ environment variable to make sure the midnight log rotation syncs up with your timezone's midnight.
{{< /blockquote >}}

DHCPを使う場合は、ネットワーク周りに手を入れる必要がある[^2]。
私は、`network_mode: host`とする方法を採用した。

[^2]: [Docker DHCP and Network Modes - Pi-hole documentation](https://docs.pi-hole.net/docker/dhcp/)

## `etc-dnsmasq.d/`下のカスタム設定ファイル

私はDHCPでIPアドレスを配る際に、端末が使用するDNSサーバを指定している。  
これはPi-HoleのGUIから設定できないため、コンテナにマウントしている`etc-dnsmasq.d/`下に`99-custom.conf`を作成し、以下のように記述した[^3]。  
※`192.0.2.254`の部分はPi-Holeコンテナを起動しているDockerホストのIPアドレスにする  

[^3]: [Secondary DNS Server for DHCP - Help - Pi-hole Userspace](https://discourse.pi-hole.net/t/secondary-dns-server-for-dhcp/1874)

```
dhcp-option=option:dns-server,192.0.2.254,1.1.1.1,1.0.0.1
```

## `stubby.yml`  

上流のDNSサーバは1.1.1.1にしている。  

```yaml
resolution_type: GETDNS_RESOLUTION_STUB
dns_transport_list:
  - GETDNS_TRANSPORT_TLS
tls_authentication: GETDNS_AUTHENTICATION_REQUIRED
tls_query_padding_blocksize: 128
edns_client_subnet_private : 1
idle_timeout: 10000
listen_addresses:
   - 127.2.2.2@5253
   - 0::1@5253

upstream_recursive_servers:
  - address_data: 1.1.1.1
    tls_auth_name: "cloudflare-dns.com"
  - address_data: 1.0.0.1
    tls_auth_name: "cloudflare-dns.com"
  - address_data: 2606:4700:4700::1111
    tls_auth_name: "cloudflare-dns.com"
  - address_data: 2606:4700:4700::1001
    tls_auth_name: "cloudflare-dns.com"
```

## `pihole.env`  

環境に合わせてIPアドレスを記載する。  
IPv6を使わない場合、`ServerIPv6`は不要。  

```
WEBPASSWORD=hogehoge
ServerIP=192.0.2.254
ServerIPv6=2001:db8::1
```

## ディレクトリ構造  

Pi-Holeコンテナ関連ファイルの全体像（ディレクトリ構造）は以下の通り。

```bash
.
├── docker-compose.yml
├── etc-dnsmasq.d
│   └── 99-custom.conf
├── etc-pihole
├── pihole.env
└── stubby.yml
```
