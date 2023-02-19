---
title: "NeovimとDev Containerを使った開発環境の試行錯誤"
slug: neovim-devcontainer
author: ""
type: ""
date: 2023-02-19T08:19:40+09:00
draft: false
subtitle: ""
image: ""
toc: true
tocMove: true
tags: ["Neovim", "Dev Container"]
---

先日 Devbox と、今さら感はあるが VSCode による Dev Container を使ってみた。
Devbox の Dev Container の設定を生成する機能によって、ローカルの Devbox 環境を簡単に Dev Container に持ち込むことができ、とても体験がよかった。

しばらく Devbox + VSCode + Dev Container で開発をしていたが、やはり VSCode ではなく Neovim を使いたいと思い試行錯誤したのでここにまとめる。

## 環境

- OS: Pop!\_OS 22.04 LTS
- Docker 20.10.23
- Devbox 0.3.3
- Neovim v0.9.0-dev

## 挫折した方法

うまくいかなかった方法も書いておく。

### Devbox + Neovim

まず試したのが、Devbox 環境に Neovim をインストールするアプローチ。
ただ、これはうまくいかなかった。  
どうも、Devbox で使っている Nix では Neovim の設定が通常の Linux OS のように $HOME/.config/nvim に置くのではないようだ。（ちゃんと調べていない）

こちらのアプローチは早々に諦めた。

### [esensar/nvim-dev-container](https://github.com/esensar/nvim-dev-container)

最初は Devbox で生成した Dockerfile と devcontainer.json で試していたがうまくいかず。  
Devbox が生成する Dockerfile には chown コマンドを実行する部分があるのだが、nvim-dev-container でビルドする際にこの chown コマンドが無視されており、そのせいでビルドに失敗していた。

後述する Open Devcontainer でも、Devbox が生成する Dockerfile のビルドは同じ理由で失敗した。

[devcontainers/templates](https://github.com/devcontainers/templates) の Ubuntu でもビルドできなかったので、こちらのアプローチも諦めた。

## 成功した方法

Dev Container 環境で Neovim を自身の設定で動かすことに成功した方法を記す。  
動作確認は [devcontainers/templates](https://github.com/devcontainers/templates) の Ubuntu、[devcontainers/images](https://github.com/devcontainers/images) の Python で行なった。

なお、Devbox を使うことは諦めている。

### Dev Container CLI

[devcontainers/cli](https://github.com/devcontainers/cli)と、Neovim を使えるようにした Dockerfile を使う方法。

各プロジェクトでは、プロジェクトのルートにあらかじめ用意しておいた.devcontainer ディレクトリをコピーし、必要な設定を加えて利用するイメージ。

私が動作確認に使っていた Dockerfile と devcontainer.json は以下の通り。

- Dockerfile

  ```Dockerfile
  # [Choice] Ubuntu version (use ubuntu-22.04 or ubuntu-18.04 on local arm64/Apple Silicon): ubuntu-22.04, ubuntu-20.04, ubuntu-18.04
  ARG VARIANT=ubuntu-22.04
  FROM mcr.microsoft.com/vscode/devcontainers/base:0-${VARIANT}

  ARG NEOVIM_VERSION
  ARG LAZYGIT_VERSION
  ARG DELTA_VERSION

  # [Optional] Uncomment this section to install additional OS packages.
  RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
    && apt-get -y install --no-install-recommends \
    # For building neovim
    ninja-build \
    gettext \
    libtool \
    libtool-bin \
    autoconf \
    automake \
    cmake \
    g++ \
    pkg-config \
    unzip \
    curl \
    doxygen \
    lua5.3 \
    # mason.nvim may require
    nodejs \
    npm \
    python3 \
    python3-pip \
    # utility
    ripgrep \
    fish \
    # Lazygit
    && mkdir -p /tmp/lazygit \
    && cd /tmp/lazygit \
    && curl -fsSL https://github.com/jesseduffield/lazygit/releases/download/v${LAZYGIT_VERSION}/lazygit_${LAZYGIT_VERSION}_Linux_x86_64.tar.gz | tar xz \
    && install lazygit /usr/local/bin \
    # Neovim
    && mkdir -p /tmp/ \
    && cd /tmp \
    && git clone https://github.com/neovim/neovim \
    && cd /tmp/neovim \
    && git checkout ${NEOVIM_VERSION} \
    && make -j4 \
    && make install \
    # delta
    && mkdir -p /tmp/delta \
    && cd /tmp/delta \
    && curl -fsSL https://github.com/dandavison/delta/releases/download/${DELTA_VERSION}/delta-${DELTA_VERSION}-x86_64-unknown-linux-musl.tar.gz | tar xz --strip-components 1 \
    && mv /tmp/delta/delta /usr/local/bin/ \
    # cleanup
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /tmp/*
  ```

- devcontainer.json

  ```json
  {
    "name": "Neovim",
    "build": {
      "dockerfile": "Dockerfile",
      // Update 'VARIANT' to pick an Ubuntu version: jammy / ubuntu-22.04, focal / ubuntu-20.04, bionic /ubuntu-18.04
      // Use ubuntu-22.04 or ubuntu-18.04 on local arm64/Apple Silicon.
      "args": {
        "VARIANT": "jammy",
        "NEOVIM_VERSION": "nightly",
        "LAZYGIT_VERSION": "0.37.0",
        "DELTA_VERSION": "0.15.1"
      }
    },
    "features": {
      "ghcr.io/devcontainers/features/sshd:1": {}
    },
    "appPort": ["127.0.0.1:2222:2222"],

    // Use 'forwardPorts' to make a list of ports inside the container available locally.
    // "forwardPorts": [],

    // Use 'postCreateCommand' to run commands after the container is created.
    // "postCreateCommand": "uname -a",

    // Comment out to connect as root instead. More info: https://aka.ms/vscode-remote/containers/non-root.
    "remoteUser": "vscode",
    "mounts": [
      "source=${localEnv:HOME}/.config/nvim,target=/home/vscode/.config/nvim,type=bind"
    ]
  }
  ```

Neovim の設定は、ローカルのものをコンテナにマウントする。

Dev Container CLI にはコンテナにアタッチする機能がないため、SSH を使ってコンテナ内に入る。
SSH の設定には Dev Container Features を使っている。

また、Dev Container CLI にはまだポートフォワード機能が実装されていないため、 appPort を使っている。

SSH のように Dev Container Features を使うか、プロジェクトにコピーした Dockerfile に追加することでプロジェクト固有のツールや設定を足すことができる。

コンテナ内に SSH 接続するスクリプトは、[こちら](https://github.com/devcontainers/cli/tree/main/example-usage/tool-vim-via-ssh)を参考にした。  
後述するリポジトリに含めているので参照いただきたい。

### Open Devcontainer

[Open Devcontainer](https://gitlab.com/smoores/open-devcontainer) は Dev
Container の仕様に準拠しつつ、独自の機能が追加されている。  
その 1 つが指定した dotfiles リポジトリの自動インストール機能である。

この方法では、この自動インストール機能を利用する。

Dec Container
CLI を使う方法では、Neovim が入ったコンテナイメージにプロジェクト固有のツールや設定を追加するが、こちらでは各プロジェクトのコンテナイメージに Neovim とその設定を足すイメージ。

ポイントは、インストールスクリプトを `install` という名前で dotfiles リポジトリのルートに置き、実行権限をつけておくこと。

その上で、 `odc run --dotfiles-repo <url of dotfiles repository>`
と実行すると、devcontainer.json で指定したイメージや Dockerfile からコンテナを作成したあとに、 `install` スクリプトを実行してくれる。

リポジトリの URL を以下のように $HOME/.config/odc.json に記述しておくと、 `odc run`
のオプションを省略できる。

```json
{
  "dotfiles-repo": "<url of dotfiles repository>"
}
```

## Pros and Cons

### Dev Container CLI を使う方法のメリット

- Dev Container Features を利用できる
  - プロジェクトによっては、devcontainer.json に Features を追加するだけでよい

### Dev Container CLI を使う方法のデメリット

- そのままではローカルの SSH 設定や GPG を利用できない
- 現時点では起動したコンテナを Dev Container CLI から停止できない

### Open Devcontainer を使う方法のメリット

- SSH と GPG のフォワーディング機能がある
  - コンテナ内からローカルの SSH 設定と GPG 鍵を利用できる
- dotfiles リポジトリの自動インストール機能がある
- 起動したコンテナの停止コマンドが実装されている

### Open Devcontainer を使う方法のデメリット

- 現時点では Dev Container Features を利用できない
- プロジェクトごとに Dockerfile と devcontainer.json を作成する必要がある

## さいごに

上記成功した方法に必要なファイルを含む Neovim の設定リポジトリを GitHub にホストした。

[bridge-y/nvim: Neovim Configuration](https://github.com/bridge-y/nvim)

Open Devcontainer を使う方法が気に入っているので、いまはこちらを使っている。  
ちなみに、この取り組み中に Neovim の設定を lua 化した。
