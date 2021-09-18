---
title: "SimpleLoginのカスタムドメイン設定"
slug: custom-domain-on-simplelogin
author: ""
type: ""
date: 2021-09-18T23:59:23+09:00
draft: false
subtitle: ""
image: ""
toc: false
tocMove: false
tags: ["SimpleLogin", "Domain"]
---

[SimpleLogin](https://simplelogin.io/)に取得したドメインを設定したので、手順をメモ。

SimpleLoginのWebページに取得したドメインを入力したら、あとは表示される値をDNSレコードにコピペするだけなので非常に楽だった。
取得したドメインを管理しているサイトのUIによっては面倒かもしれない。

なお、下記手順は2021/09/07時点のものである。

1. SimpleLoginのダッシュボードを開く
2. `Domains`をクリックする
3. `New Domain`にドメインを入力する
4. `Create`をクリックする
5. 指示に従い、ドメインを取得したレジストラでTXTレコードを追加する
6. 指示に従い、ドメインを取得したレジストラでMXレコードを追加する
7. (Optional) 指示に従い、ドメインを取得したレジストラでSPFの設定をする
8. (Optional) 指示に従い、ドメインを取得したレジストラでDKIMの設定をする
9. (Optional) 指示に従い、ドメインを取得したレジストラでDMARCの設定をする
