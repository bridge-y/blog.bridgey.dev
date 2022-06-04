---
title: "Docker SwarmのComposeファイルで環境変数を使う"
slug: use-environment-variables-in-docker-stack
author: ""
type: ""
date: 2022-06-04T16:30:11+09:00
draft: false
subtitle: ""
image: ""
toc: false
tocMove: false
tags: ['Docker', 'Docker Swarm']
---

Docker Composeでは同じフォルダの`.env`ファイルを読み込んでくれる。
Composeファイル中に変数を使いたいときにこの機能はとても便利。  
しかし、Swarmにはこの機能がない。  

{{< blockquote author="Environment variables in Compose | Docker Documentation" link="https://docs.docker.com/compose/environment-variables/" >}}
The .env file feature only works when you use the docker-compose up command and does not work with docker stack deploy.
{{< /blockquote >}}

[^1]: 私の場合、Traefikのルーティング設定に使うドメインをComposeファイルにベタ書きしたくなかった。その他にも、環境や用途によって値を変えたい場合など、様々な理由から変数化したいケースがある。

Swarmでも`.env`ファイルを使いたいケース[^1]があるため、備忘録として記録しておく。

## 起動にMakefileを使う

このようなMakefileを用意し、makeコマンドで起動するようにする。

```
stack = example
envfile = ./.env

include $(envfile)
export

up:
	docker stack deploy -c docker-stack.yml --resolve-image always $(stack)

down:
	docker stack rm $(stack)

config:
	docker compose -f docker-stack.yml config
```

こうすると、Docker SwarmでもComposeファイルに変数を使い、`.env`ファイルで値を設定できる。  
`envfile = ./.env`部分を置き換えることで任意のファイル名 (`example.env`など) にすることも可能。  
`envfile = ./foo.env ./bar.env`のように複数ファイルを設定することもできる。

SwarmのStackコマンドは長くなりがち(個人的な感想)なので、コマンドを短くできるのも利点。

私は`make up`でStackをデプロイ、`make down`でStackを削除するように設定している。  
`make config`はComposeファイル中の変数が意図通りに置き換えられるか確認するために使用している。
