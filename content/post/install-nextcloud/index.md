---
title: "自宅サーバにNextcloudを設置した"
author: ""
type: ""
date: 2020-04-07T20:05:16+09:00
draft: false
subtitle: ""
image: ""
toc: true
tocMove: true
tags: ["Nextcloud", "Docker"]
---

家族が写真や音楽データを保存できるように、自宅サーバに[Nextcloud](https://nextcloud.com/)をインストールしました。

## きっかけ

妻は一眼レフで写真を撮るのですが、PCのストレージがいっぱいになり、保存用のストレージを欲していました。  
クラウドストレージよりは自宅で保管したいとのことなので、私のサーバにNextcloudを導入し、そこに保存してもらうことにしました。  
Nextcloudを選んだ理由は、Androidアプリもあり、使いやすいと考えたためです。

以前ブログに書いたHDDの増設[^1]や、バックアップ[^2]はこのための準備でした。  

[^1]: [btrfsでHDDを増設してRAID1に変換する]({{<ref "/post/btrfs-raid1/index.md">}})
[^2]: [Btrbkを使ったバックアップ]( {{<ref "/post/btrfs-raid1/index.md">}} )

## インストール

Dockerを使ってインストールしました。  

- 環境
  - OS: Ubuntu 18.04.4 LTS
  - Docker Engine: version 19.03.8
  - docker-compose: version 1.25.4
- Composeファイル
```yaml
version: '3.7'

services:
  db:
    image: linuxserver/mariadb
    init: true
    restart: unless-stopped
    volumes:
      - ./data/db:/config
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Asia/Tokyo
    env_file:
      - ./db.env

  redis:
    image: redis:alpine
    init: true
    restart: unless-stopped
    volumes:
      - ./data/redis:/data

  app:
    image: nextcloud
    ports:
      - 8080:80
    depends_on:
      - db
      - redis
    volumes:
      - ./data/nextcloud:/var/www/html
      - ./php.ini:/usr/local/etc/php/conf.d/zzz-custom.ini
    init: true
    restart: unless-stopped
    environment:
      MYSQL_HOST: db:3306
      REDIS_HOST: redis
    env_file:
      - ./db.env
```
MariaDBは公式のイメージを使おうとしましたが、パーミッションの影響でマウントしたホストのディレクトリに書き込めないという問題に直面し、面倒くさくなって[LinuxServer.io](https://www.linuxserver.io/)のイメージに替えました。  
ただ、LinuxServer.ioのイメージは公式イメージのように設定を引数で与えられない(ちゃんと調べていません)ようなので、そこが少しイマイチと感じています。  
- MariaDB 設定ファイル   
  [Nextcloudのドキュメント](https://docs.nextcloud.com/server/15/admin_manual/configuration_database/linux_database_configuration.html)の設定をそのまま使っています。  
  MariaDBにマウントしたディレクトリ(./data/db/)にcustom.cnfとして保存しています。  
```
[server]
skip-name-resolve
innodb_buffer_pool_size = 128M
innodb_buffer_pool_instances = 1
innodb_flush_log_at_trx_commit = 2
innodb_log_buffer_size = 32M
innodb_max_dirty_pages_pct = 90
query_cache_type = 1
query_cache_limit = 2M
query_cache_min_res_unit = 2k
query_cache_size = 64M
tmp_table_size= 64M
max_heap_table_size= 64M
slow-query-log = 1
slow_query_log_file	= /config/log/mysql/mariadb-slow.log
long_query_time = 1

[client-server]
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mariadb.conf.d/

[client]
default-character-set = utf8mb4

[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_general_ci
transaction_isolation = READ-COMMITTED
binlog_format = ROW
innodb_large_prefix=on
innodb_file_format=barracuda
innodb_file_per_table=1
```
- db.env  
```
MYSQL_ROOT_PASSWORD=dummypassword
MYSQL_PASSWORD=dummypassword
MYSQL_DATABASE=nextcloud
MYSQL_USER=nextcloud
```
- php.ini  
```
max_execution_time = 3600
upload_max_filesize = 4g
memory_limit = 4g
```

他にも内向きDNSサーバやリバースプロキシの設定をしていますが、今回は割愛します。

### Nextcloud上でインストールしたアプリ

- [Camera RAW Previews](https://apps.nextcloud.com/apps/camerarawpreviews)  
  一眼レフのRAW画像をプレビューできるようにするため。
- [Music](https://apps.nextcloud.com/apps/music)  
  音楽データをNextcloud上で再生できるようにするため。
- [Keeweb](https://apps.nextcloud.com/apps/keeweb)  
  子どもの口座情報や認証情報を共有するため。
- [Contacts](https://apps.nextcloud.com/apps/contacts)  
  家族の連絡先を共有するため。

Music、Keeweb、Contactsはインストールすると、UIの上部にアイコンが現れます。  
{{<figure src="ui.png" caption="一番左はNextcloudのロゴ。ロゴの右にあるアイコンは、左から、ファイル、写真、アクティビティ、Keeweb、連絡先 (Contacts)、ミュージック (Music)。">}}

## 我が家の用途

前述の通り、主に妻がスマートフォンや一眼レフの写真のバックアップ用途で使っています。  
私は[pCloud](https://www.pcloud.com/)の2TBのLifetimeプランを購入していることもあり、写真や音楽データはもっぱらpCloudに保存していて、Nextcloudはもともと使うつもりはありませんでした。  
趣味としてNextcloudやDockerで遊ぶだけのつもりでした。  

しかし、最近は[プラグイン](#プラグイン)で書いたように、子どもの口座情報のような共有しておいた方がいい情報をNextcloudに保存するようになったので、今後はもっと活用したいと思っています。  

## おわりに

Nextcloudによって端末のストレージの容量不足問題から解消されました。  
ブラウザからも、スマートフォンのアプリからも、Dropboxのようなクラウドストレージと同等の使い勝手です。  
NASを買うという案もありましたが、せっかくなら楽しそうな方で、ということでNextcloudを採用しました。  
今のところ、妻も満足しているようなのでよかったです。  

ただ、大量の画像があるフォルダにアクセスすると挙動が怪しくなるようです。  
おそらくプレビュー関連の問題だと思うので、近いうちに対応したいと思います。  

