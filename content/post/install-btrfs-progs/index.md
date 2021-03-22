---
title: "btrfs-progsの最新版をインストール"
slug: install-latest-btrfs-progs
author: ""
type: ""
date: 2020-02-11T15:47:50+09:00
draft: false
subtitle: ""
image: ""
tags: ["btrfs"]
---

ノートPCにOSを入れ直したついでに、カーネルのバージョンも上げたら`btrfs send`が動作しなくなりました。  
最新版btrfs-progsのインストール手順を自分用に残しておきます。  

# 環境
OS: Linux Mint 19.3 MATE 64-bit  
Kernel: 5.3.0-28-generic  
btrfs-progs: v4.15.1  

上記環境だと`btrfs send`が動作しなかった。

# 必要なソフトウェアのインストール
```
$ sudo apt install git asciidoc xmlto --no-install-recommends
$ sudo apt install python3.6-dev
$ sudo apt install uuid-dev libattr1-dev zlib1g-dev libacl1-dev e2fslibs-dev libblkid-dev liblzo2-dev libzstd-dev
```
# 最新版btrfs-progsのインストール

`configure`でエラーが出たので、`--disable-convert`オプションを付けた。  

```
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/kdave/btrfs-progs.git
$ cd btrfs-progs/
$ ./autogen.sh
$ ./configure --disable-convert  # reiserfs/misc.hがないというエラーが出たので
$ make
$ sudo make install
```

# バージョン確認

```
$ btrfs version
btrfs-progs v5.4.1
```
# 注意点

Homebrewの環境変数が設定されているとコンパイルに失敗した。  
面倒なので、Homebrew環境変数を設定していない状態でコンパイルしたほうがよい。

# おわりに

最新版を使うことで無事に`btrfs send`が動作しました。  
これでやっとバックアップできます。  

## 参考ページ
[How To Build and Install the Latest Version of btrfs-tools on Linux - nixCraft](https://www.cyberciti.biz/faq/how-to-build-and-install-the-latest-version-of-btrfs-tools-on-linux/)

