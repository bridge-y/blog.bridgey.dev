---
title: "jqコマンドで値をキーとしたオブジェクトに整形する方法"
slug: how-to-convert-value-to-key-using-jq
author: ""
type: ""
date: 2021-03-04T09:29:51+09:00
draft: false
subtitle: ""
image: ""
toc: false
tocMove: false
tags: ['jq']
---

ひと工夫必要だったのでメモ。

## やりたいこと

jqコマンドを使って値をキーとしたオブジェクトに整形したい。具体例を挙げると以下のようなイメージ。

```json
{
  "name": "taro",
  "age": 20,
  "info": [
    "hoge",
    "fuga"
  ]
}
```

上のようなJSONを、下のように加工したい。

```json
{
  "taro": [
    "hoge",
    "fuga"
  ]
}
```

## jqでは取得した値をオブジェクトのキーにできない

検証環境は次の通り。

```
$ jq -V
jq-1.6
```

単純に取得した値をキーとすれば良さそうだが、エラーとなってしまう。

```bash
$ echo '{"name":"taro","age":20,"info":["hoge", "fuga"]}' | jq '. | {.name: .info}'
jq: error: syntax error, unexpected FIELD (Unix shell quoting issues?) at <top-level>, line 1:
. | {.name: .info}
jq: error: May need parentheses around object key expression at <top-level>, line 1:
. | {.name: .info}
jq: 2 compile errors
```

## 解決方法

キーとしたい値を使って適当なオブジェクトを作り、そこから値を取得する。  

```bash
$ echo '{"name":"taro","age":20,"info":["hoge", "fuga"]}' | jq '. | {({name: .name} | to_entries[] | .value): .info}'
{
  "taro": [
    "hoge",
    "fuga"
  ]
}
```

次のような書き方もできる。

```bash
$ echo '{"name":"taro","age":20,"info":["hoge", "fuga"]}' | jq '. | .name as $name | {({$name} | to_entries[] | .value): .info}'
{
  "taro": [
    "hoge",
    "fuga"
  ]
}
```

## 参考にしたページ

[jq Manual (development version)](https://stedolan.github.io/jq/manual/)
