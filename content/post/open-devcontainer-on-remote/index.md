---
title: "SSH接続先のOpen DevcontainerでSSH接続元のgpg-agentを使う"
slug: open-devcontainer-on-remote
author: ""
type: ""
date: 2023-03-18T18:42:20+09:00
draft: false
subtitle: ""
image: ""
toc: false
tocMove: false
tags: ["Dev Container"]
---

[Open Devcontainer](https://gitlab.com/smoores/open-devcontainer) を SSH 接続先で使う際に、SSH 接続元の GPG エージェントを使用するための設定メモ。

## 要約

- Open Devcontainer は agent-extra-socket をコンテナに転送するという仕様になっている
- SSH の Unix ドメインソケット転送で、接続元の agent-extra-socket を接続先の agent-socket に転送する
- SSH 接続先で、agent-socket から agent-extra-socket にリンクを張る

## Open Devcontainer の仕様

Open Devcontainer がどのようにコンテナに gpg-agent を転送しているかソースコードを確認したところ、ローカルの agent-extra-socket をコンテナに転送していることがわかった[^1]。

[^1]: https://gitlab.com/smoores/open-devcontainer/-/blob/main/cmd/forwarddevtools.go

## GPG エージェントの転送設定

- ローカル（SSH 接続元）の `~/.ssh/config` で GPG エージェントを転送したいホストの設定に `RemoteForward <agent-socket_on_remote> <agent-extra-socket_on_local>` を記載する

  - \<agent-socket_on_remote\>: SSH 接続**先**で `gpgconf --list-dir agent-socket` を実行して得られた値
  - \<agent-extra-socket_on_local\>: SSH 接続**元**で `gpgconf --list-dir agent-extra-socket` を実行して得られた値
  - こんな感じ

    ```
    Host host.example.com
      RemoteForward <agent-socket_on_remote> <agent-extra-socket_on_local>
    ```

- リモート（SSH 接続先）の `/etc/ssh/sshd_config` に `StreamLocalBindUnlink yes` を追記する

- リモート（SSH 接続先）の systemd で gpg が起動しないようにする

  - 設定ファイルに自動起動を無効化する設定を追記

    ```bash
    $ cat ~/.gnupg/gpg.conf
    no-autostart
    ```

  - systemd による自動起動を無効化

    ```bash
    systemctl --user mask gpg-agent.service gpg-agent.socket gpg-agent-ssh.socket gpg-agent-extra.socket gpg-agent-browser.socket
    ```

## SSH 接続先で agent-extra-socket にリンクを張る

リモート（SSH 接続先）で以下のコマンドを実行する。

```bash
ln -sf $(gpgconf --list-dir agent-socket) $(gpgconf --list-dir agent-extra-socket)
```

## 課題（自分の環境だけ？）

ローカルで Open Devcontainer を使った際、起動したコンテナ上で GPG を使えていない。

リモートで使用する設定を行なう前は使えていた気がする（記憶が朧げ）が、原因を特定できていない。

## 参考

- [AgentForwarding - GnuPG wiki](https://wiki.gnupg.org/AgentForwarding)
- [GPG Agent Forwarding - Qiita](https://qiita.com/kino-ma/items/6aab9ad7605e5a25b962)
- [gpg-agent を ssh で forwarding する設定を改善した - @znz blog](https://blog.n-z.jp/blog/2022-10-14-gpg-agent-forwarding.html)
