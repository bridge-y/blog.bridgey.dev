---
title: "NFSサーバとDocker Swarmノードを兼用している環境でNFSボリュームを使う際の注意"
author: ""
type: ""
date: 2021-03-02T22:02:13+09:00
draft: false
subtitle: ""
image: ""
toc: false
tocMove: false
tags: ['docker', 'nfs']
---

NFSサーバとDocker Swarmノードを兼用している場合、自身のIPアドレスに対しても共有する設定が必要だったのでメモ。

## 環境

2台でSwarmクラスタを組んでいる。そのうちの1台はNFSサーバも兼用している。

- ホストA (192.0.2.10) : Swarmマネージャー、NFSサーバ
- ホストB (192.0.2.20) : Swarmワーカー

当初ホストAでは、以下のようにホストBに対してだけ共有する設定にしていた。

```
# /etc/exportsの設定
/hoge 192.0.2.20(rw,sync,no_subtree_check)
/fuga 192.0.2.20(rw,sync,no_subtree_check)
```

## 事象

上記の設定でDocker SwarmでNFSボリュームを使おうとしたところ、コンテナがホストAに配置された場合にマウントできずエラーとなった。
このとき、ホストBにコンテナが配置された場合はマウントできていた。

以下は使用していた`docker-stack.yml`の疑似ファイルとStack作成の疑似コマンド。

```
version: '3.7'

services:
  foo:
    image: example
    init: true
    volumes:
      - hoge:/hoge
      - fuga:/fuga
    deploy:
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
volumes:
  hoge:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.0.2.10,nolock,soft,rw
      device: ":/hoge"
  fuga:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.0.2.10,nolock,soft,rw
      device: ":/fuga"
```

```
docker stack deploy -c docker-stack.yml example
```

## 対応

ホストAの`/etc/exports`に、ホストAのIPアドレス (`localhost`ではダメだった) に対する設定を追記することで解決した。

```
# /etc/exportsの設定
/hoge 192.0.2.20(rw,sync,no_subtree_check) 192.0.2.10(rw,sync,no_subtree_check)
/fuga 192.0.2.20(rw,sync,no_subtree_check) 192.0.2.10(rw,sync,no_subtree_check)
```
