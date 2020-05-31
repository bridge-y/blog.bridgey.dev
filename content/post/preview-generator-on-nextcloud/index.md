---
title: "Preview GeneratorでNextcloudのサムネイル生成"
author: ""
type: ""
date: 2020-05-31T18:26:29+09:00
draft: false
subtitle: ""
image: ""
toc: true
tocMove: true
tags: ["Nextcloud", "Docker"]
---

我が家ではNextcloudを運用しています。  
(前回の記事: [自宅サーバにNextcloudを設置した]({{<ref "/post/install-nextcloud/index.md">}}))

Nextcloudでフォルダに多数の画像ファイルを保存していると、そのフォルダにアクセスした時に処理が重くなる現象が発生していました。  
Nextcloudのデフォルト設定では、フォルダにアクセスした時にサムネイルの生成処理が走ることが、処理の重さの原因のようです[^1]。

[^1]: [Thumbnail generator - support - Nextcloud community](https://help.nextcloud.com/t/thumbnail-generator/3035)

そこで、[Preview Generator](https://apps.nextcloud.com/apps/previewgenerator)を導入して前もってサムネイルを作成しておくことにしました。  
備忘録として、設定内容を残しておきます。


## プレビューの最大サイズを変更

Dockerを使用している場合は、まずコンテナ内に入ります。

```
php occ config:system:set preview_max_x --value 1080
php occ config:system:set preview_max_y --value 1920
```

## 生成するサムネイルのサイズを指定

```
php ./occ config:app:set --value="32 64 256 1024 1920"  previewgenerator squareSizes
php ./occ config:app:set --value="64 128 1024 1080 2030" previewgenerator widthSizes
php ./occ config:app:set --value="64 256 1024 1080 1920" previewgenerator heightSizes
```

## Preview Generatorを実行

以下のコマンドでサムネイルを一括生成する。

```
php ./occ preview:generate-all
```

### 注意点

上記コマンドを実行すると、対象ファイルの数によってはメモリの使用量が大きくなります。  
Dockerを使っていて、Nextcloudのコンテナの使用メモリ量を制限している場合は途中で終了してしまうため、要注意です。  
私は2GBに設定していたため、途中で終了してしまいました。  

## cronに登録

定期的にサムネイルを生成するために、Preview Generatorの実行をcronに登録する。

### Nextcloudのバックグラウンドジョブの設定をcronに変更

デフォルトではAJAXになっているので、これをcronに変更する。  
管理者権限を持つアカウントでNextcloudにログイン後、設定ページの**基本設定**にアクセスすると、**バックグラウンドジョブ**という項目があるので、そこから変更できる。

### cron用のコンテナを用意する

Dockerを使用している場合、Dockerホストのcronに登録する方法と、定期実行用のcron用コンテナを用意する方法がある。  
私はcron用コンテナを用意したので、その方法について記載します。  

[GitHub - rcdailey/nextcloud-cronjob: A simple container to run cron.php in your Nextcloud docker container](<https://github.com/rcdailey/nextcloud-cronjob>)を利用しました。  
なお、Docker Swarmを使っている場合は上記コンテナは正しく動きません。  
（私はDocker SwarmでNextcloudを実行しています。前回の記事のあとに、Swarmに移行していました。）  
次のように修正することで動作するようになりました。  

```
diff --git a/Dockerfile b/Dockerfile
index cea19f3..ed21e86 100644
--- a/Dockerfile
+++ b/Dockerfile
@@ -4,7 +4,7 @@ RUN apk add --no-cache docker-cli bash tini

 ENV NEXTCLOUD_EXEC_USER=www-data
 ENV NEXTCLOUD_CONTAINER_NAME=
-ENV NEXTCLOUD_PROJECT_NAME=
+ENV NEXTCLOUD_STACK_NAME=
 ENV NEXTCLOUD_CRON_MINUTE_INTERVAL=15

 COPY scripts/*.sh /
diff --git a/scripts/cron-tasks.sh b/scripts/cron-tasks.sh
index f10fe18..6a1623c 100755
--- a/scripts/cron-tasks.sh
+++ b/scripts/cron-tasks.sh
@@ -26,7 +26,9 @@ fi
 cd /cron-scripts
 for script in *.sh; do
     echo "> Running Script: $script"
-    docker exec $exec_user -i "$containerId" bash < $script
+    for cid in ${containerId}; do
+        docker exec $exec_user -i $cid bash < $script
+    done
 done

 echo "> Done"
diff --git a/scripts/find-container.sh b/scripts/find-container.sh
index ecce0f9..83e38db 100755
--- a/scripts/find-container.sh
+++ b/scripts/find-container.sh
@@ -1,15 +1,11 @@
 #!/usr/bin/env bash
 set -e

-if [[ ! -z "$NEXTCLOUD_PROJECT_NAME" ]]; then
-    containerName="${NEXTCLOUD_PROJECT_NAME}_"
-else
-    matchEnd=","
+if [[ ! -z "$NEXTCLOUD_STACK_NAME" ]]; then
+    containerName="${NEXTCLOUD_STACK_NAME}_"
 fi

 containerName="${containerName}${NEXTCLOUD_CONTAINER_NAME}"

 # Get the ID of the container so we can exec something in it later
-docker ps --format '{{.Names}},{{.ID}}' | \
-    egrep "^${containerName}${matchEnd}" | \
-    awk '{split($0,a,","); print a[2]}'
+docker ps -q -f name=${containerName}
diff --git a/scripts/healthcheck.sh b/scripts/healthcheck.sh
index d5f877b..a2b6180 100755
--- a/scripts/healthcheck.sh
+++ b/scripts/healthcheck.sh
@@ -5,4 +5,4 @@ set -x
 ps -o comm | grep crond || exit 1

 # Make sure the target container is still running/available
-docker inspect -f '{{.State.Running}}' "$(/find-container.sh)" | grep true || exit 1
+/find-container.sh | xargs -i docker inspect -f '{{.State.Running}}' {} | grep true || exit 1
```

cronに関するdocker-compose.ymlは以下のとおりです。

```
# for cron
  cron:
    build:
      context: ./nextcloud-cron/
    image: my-nextcloud-cron
    networks:
      - backend
    depends_on:
    - app
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
    - /etc/localtime:/etc/localtime:ro
    - ./run-previewgenerator-pre-generate.sh:/cron-scripts/run-previewgenerator-pre-generate.sh:ro
    environment:
    - NEXTCLOUD_CONTAINER_NAME=app
    - NEXTCLOUD_STACK_NAME=nextcloud
    deploy:
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
```

`run-previewgenerator-pre-generate.sh`には、以下のコマンドを記述しています。  

```
#!/usr/bin/env bash
./occ preview:pre-generate
```

## （参考）Docker Swarm用docker-compose.yml全体

custom.confやphp.iniの内容は前回の記事を参照してください。  

```
version: '3.7'

services:
  db:
    image: linuxserver/mariadb
    networks:
      - backend
    init: true
    volumes:
      - ./data/db:/config
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Asia/Tokyo
    env_file:
      - ./db.env
    deploy:
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      resources:
        limits:
          memory: 4g

  redis:
    image: redis:alpine
    networks:
      - backend
    init: true
    volumes:
      - ./data/redis:/data
    deploy:
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      resources:
        limits:
          memory: 1g

app:
    image: nextcloud
    networks:
      - swarm_net
      - backend
    ports:
      - 8080:80
    depends_on:
      - db
      - redis
    volumes:
      - ./data/nextcloud:/var/www/html
      - ./php.ini:/usr/local/etc/php/conf.d/zzz-custom.ini
    init: true
    environment:
      MYSQL_HOST: db:3306
      REDIS_HOST: redis
    env_file:
      - ./db.env
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      resources:
        limits:
          memory: 4g

  # for cron
  cron:
    build:
      context: ./nextcloud-cron/
    image: my-nextcloud-cron
    networks:
      - backend
    depends_on:
    - app
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
    - /etc/localtime:/etc/localtime:ro
    - ./run-previewgenerator-pre-generate.sh:/cron-scripts/run-previewgenerator-pre-generate.sh:ro
    environment:
    - NEXTCLOUD_CONTAINER_NAME=app
    - NEXTCLOUD_STACK_NAME=nextcloud
    deploy:
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure

networks:
  backend:
  swarm_net:
    driver: overlay
    external: true
```

## 参考ページ

[Improving Nextcloud's Thumbnail Response Time - www.bentasker.co.uk](<https://www.bentasker.co.uk/documentation/linux/671-improving-nextcloud-s-thumbnail-response-time>)
