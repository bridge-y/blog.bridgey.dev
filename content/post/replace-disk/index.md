---
title: "RAID1 Btrfsの壊れかけたHDDを交換した"
slug: replace-hdd-on-btrfs-raid1
author: ""
type: ""
date: 2021-04-01T21:42:46+09:00
draft: false
subtitle: ""
image: ""
toc: true
tocMove: false
tags: ['Btrfs']
---

## まえがき

我が家のサーバでは、BtrfsでRAID1を組んでいます。  
ある日サーバのdmesgのメッセージを眺めていたら不穏な文字を発見しました。

```bash
[  213.455229] BTRFS info (device sda): read error corrected: ino 0 off 2584293048320 (dev /dev/sdd1 sector 1306005312)
[  213.455653] BTRFS info (device sda): read error corrected: ino 0 off 2584293052416 (dev /dev/sdd1 sector 1306005320)
[  213.456017] BTRFS info (device sda): read error corrected: ino 0 off 2584293056512 (dev /dev/sdd1 sector 1306005328)
[  213.456293] BTRFS info (device sda): read error corrected: ino 0 off 2584293060608 (dev /dev/sdd1 sector 1306005336)
[  213.606849] BTRFS error (device sda): parent transid verify failed on 2584401199104 wanted 490027 found 358635
[  213.613914] BTRFS info (device sda): read error corrected: ino 0 off 2584401199104 (dev /dev/sdd1 sector 1306216544)
[  213.614317] BTRFS info (device sda): read error corrected: ino 0 off 2584401203200 (dev /dev/sdd1 sector 1306216552)
[  213.614811] BTRFS info (device sda): read error corrected: ino 0 off 2584401207296 (dev /dev/sdd1 sector 1306216560)
[  213.615086] BTRFS info (device sda): read error corrected: ino 0 off 2584401211392 (dev /dev/sdd1 sector 1306216568)
[  213.634683] BTRFS error (device sda): parent transid verify failed on 2584406474752 wanted 490027 found 253359
```

また、`btrfs device stats`を実行したところ、こちらでもエラーが記録されていました。  

```bash
sudo btrfs device stats /mnt/raid/
# /dev/sdd1以外は省略
[/dev/sdd1].write_io_errs    689697
[/dev/sdd1].read_io_errs     375188
[/dev/sdd1].flush_io_errs    8609
[/dev/sdd1].corruption_errs  0
[/dev/sdd1].generation_errs  0
```

少し前に[同じような状況でHDDを交換したというエントリ](https://blanktar.jp/blog/2020/06/btrfs-replace-hdd)を読んでいたので、完全に壊れる前にHDDを交換することにしました。

## 環境

Ubuntu 18.04.5 LTS  
btrfs-progs v5.11

## 交換対象のHDDの物理的な特定

交換処理を始める前に、故障したHDDがどこに接続されているか確認しておきます。  
私の場合複数のHDDを使っているので、間違えないように確認しておくことにしました。  

以下のコマンドでS.M.A.R.T.情報を確認し、シリアルナンバーを把握します。  
インストールされていない場合は、`sudo apt install smartmontools`でインストールします。

```bash
sudo smartctl /dev/sdd -i
smartctl 6.6 2016-05-31 r4324 [x86_64-linux-5.4.0-67-generic] (local build)
Copyright (C) 2002-16, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Device Model:     ST4000VN008-2DR166
Serial Number:    ZM40GTSX
LU WWN Device Id: 5 000c50 0c48efad2
Firmware Version: SC60
User Capacity:    4,000,787,030,016 bytes [4.00 TB]
Sector Sizes:     512 bytes logical, 4096 bytes physical
Rotation Rate:    5980 rpm
Form Factor:      3.5 inches
Device is:        Not in smartctl database [for details use: -P showall]
ATA Version is:   ACS-3 T13/2161-D revision 5
SATA Version is:  SATA 3.1, 6.0 Gb/s (current: 6.0 Gb/s)
Local Time is:    Fri Mar 19 23:12:22 2021 JST
SMART support is: Available - device has SMART capability.
SMART support is: Enabled
```

実行結果のSerial Numberの値と、HDD表面のシリアルナンバーを比較することで、故障したHDDを特定しました。

## HDDの交換

新しいHDDをサーバに接続し、データを移行します。

前回はSeagate製のHDDを購入したので、今回はWestern Digital製にしてみました。
ついでに容量アップを図っています（4TB→6TB）。

新しいサーバのデバイス名を確認後、以下のコマンドを実行し、replaceを開始しました。  
`/dev/sde`が新しいHDDです。

```bash
sudo btrfs replace start /dev/sdd1 /dev/sde /mnt/raid/
```

{{< tweet user="bridgement" id="1373316358331457537" >}}  

1日に3%しか進まなくて絶望していましたが、1週間経って22％くらいになってからは、1日に20%進むようになりました。

```bash
sudo btrfs replace status -1 /mnt/raid/
Started on 19.Mar 23:54:06, finished on 30.Mar 17:52:46, 0 write errs, 0 uncorr. read errs
```

上記はreplaceが終わった状態でのコマンドの結果です。
約11日掛かりました。

## エラーカウントのリセットとデータレイアウトのバランス

[まえがき](#まえがき)に記載したエントリと同様に、エラーカウントのリセットとデータレイアウトのバランスを実行します。  

- エラーカウントのリセット 

  ```bash
  sudo btrfs device stats --reset /mnt/raid/
  ```

- バランス  
  容量の大きいHDDにしたので、まず容量を最大限に使用するように設定しました。  
  
    ```bash
  sudo btrfs device usage /mnt/raid/  # resizeコマンドで使用するIDの確認
  sudo btrfs filesystem resize 3:max /mnt/raid/
  ```

  その後、balanceを実行し、データの偏りを均してあげました。  

  ```bash
  sudo btrfs balance start --bg --full-balance /mnt/raid
  ```

## おわりに

完全に壊れる前に気づけてよかったです。今回気づいたのはたまたまなので、定期的にチェックする習慣を付けるか、異常を通知する仕組みを構築したいです。  

現在、絶賛バランス処理の実行中です。実行してからせっせとこのエントリを書いていて、途中経過はまだ確認していないのですが、この処理は一体どのくらい掛かるのでしょう。

## 参考

- [RAID5なbtrfsのHDDをreplaceした話 - Blanktar](https://blanktar.jp/blog/2020/06/btrfs-replace-hdd)
- [HDDのデバイス名と物理的な場所を確認: パソコン鳥のブログ](https://vogel.at.webry.info/201706/article_7.html)

