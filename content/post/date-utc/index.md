---
title: "dateコマンドで時刻をUTCに変換する"
author: ""
type: ""
date: 2019-10-06T16:44:50+09:00
subtitle: ""
image: ""
tags: ["shell"]
---


時刻をUTCでしか扱えないプログラムありますよね。
JSTをUTCに変換してから入力しないといけないのが面倒くさい｡

dateコマンドでも変換できることを知ったので、備忘録として記事にしました｡


# 概要

`2019-10-06T16:00:00+09:00` ← こういうタイムゾーン指定子が付いた表記であれば、`date`コマンドでUTCに変換できます。

# 方法

`--date`オプションにタイムゾーン指定子付きの時刻を渡し、`-u`または`--universal`オプションを付ける。

```
$ date --date "2019-10-06T16:00:00+09:00" --universal
2019年 10月  6日 日曜日 07:00:00 UTC
$ date --date "2019-10-06T16:00:00" -u  # タイムゾーン指定子がないと変換できない
2019年 10月  6日 日曜日 16:00:00 UTC
```

# その他

ISO8601形式で表示する`-I`オプション（特に`-Iseconds`）が便利。

```
$ date --date "2019-10-06T16:00:00+09:00" -I -u  # ただの-Iオプションではタイムゾーン指定子が消えてしまう
2019-10-06
$ date --date "2019-10-06T16:00:00+09:00" -Iseconds -u
2019-10-06T07:00:00+00:00
```


## 検証環境

```
$ date --version
date (GNU coreutils) 8.25
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Written by David MacKenzie.
```

