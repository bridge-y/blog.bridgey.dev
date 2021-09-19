---
title: "TailscaleネットワークでSwarmクラスタを構築"
slug: docker-swarm-with-tailscale
author: ""
type: ""
date: 2021-09-19T15:41:47+09:00
draft: false
subtitle: ""
image: ""
toc: false
tocMove: false
tags: ["Docker", "Tailscale"]
---

## Tailscaleが便利

遅まきながらTailscaleを使ってみたところ、非常に便利で感動した。

[Tailscale · Best VPN Service for Secure Networks](https://tailscale.com/)

- 設定が楽  
  インストールして認証するだけ。
- ポート開放が必要ない  
  NATを越えられるので、自宅ルータのポート開放が不要。

私はこれまで、VPSとLAN内の端末をL2で繋ぎ、VPSにVPN接続することで、外部から自宅ネットワーク内にアクセスしていた。これは、自宅ルータのポートを開放したくなかったためである。

ところが、Tailscaleのおかけで自宅ネットワークへのアクセスにVPSが必要なくなった。
ほとんどVPN接続のためにVPSを契約していたので、解約しても良かったのだが、せっかくなので自宅ネットワーク内で構築していたSwarmクラスタに組み込むことにした。

## TailscaleによるSwarmクラスタ

TailscaleのIPアドレスを使ってSwarmクラスタを組み直した。  

なお、自宅ネットワーク内で使うためのコンテナをSwarmのオーバーレイネットワークに乗せている場合は、VPSをSwarmクラスタに参加させる前に、VPSのコントロールパネルで全ポートを遮断しておいたほうがよい。  
デフォルト設定では、コンテナのポート公開がホストのiptablesの設定よりも優先されるため、何もしないと自宅用のコンテナが外部に公開されてしまう。

{{< blockquote author="Docker と iptables | Docker ドキュメント" link="https://matsuand.github.io/docs.docker.jp.onthefly/network/iptables/" >}}
FORWARDチェーンに追加されたルールは、手動によるもの、あるいは iptables ベースのファイアウォールによるものであっても、Docker のチェーンの後で評価されます。 これが何を意味するかといえば、Docker によってポートを公開したら、ファイアウォールでどのような設定を行っていても、そのポートは必ず公開されるということです。
{{< /blockquote >}}

引用したページにあるように`DOCKER-USER`チェーンにルールを追加してもよいが、VPS側のファイアウォールで全ポートを遮断する方が楽 (全ポート遮断していてもTailscaleのIPに対するSSHやSwarmの通信は届く) 。  

クラスタのノードとする端末にTailscaleをインストールし、以下のようなコマンドを実行してSwarmクラスタを組んだ。

- マネージャーノード  
  `docker swarm init --advertise-addr <自身のTailscale IP>`  
- ワーカーノード  
  `docker swarm join --token <token> <マネージャーノードのTailscale IP>:<docker port>`  
  上記コマンドはマネージャーノードで`docker swarm join-token worker`を実行することで表示される。

[以前立てたNextcloud]({{<ref "/post/install-nextcloud/index.md">}})は自宅ネットワーク内でしか使えなかったが、VPSがSwarmクラスタに加わったので、（VPSのファイアウォールでポートを開ければ）VPN接続なしでも外からアクセスできる環境になった。  
セキュリティ周りをもう少し考えてから、ドメインを設定したりリバースプロキシを立てたりして実現させたい。
